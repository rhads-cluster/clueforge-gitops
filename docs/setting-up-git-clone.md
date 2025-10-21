# Setting Up Git Cloning in InitContainer for ClueForge in RHADS

ClueForge is a Python Flask application utilizing PyTorch for generating summaries of git diff changes in OpenShift repositories. To support its git operations (cloning and pulling repositories for diff analysis), an initContainer is required to install git and clone repos into the persistent workspace (`/opt/app-root/src/workspace`). This ensures repositories like assisted-service are available on pod startup, with persistence handled by the PVC mount.

This guide documents the verified steps to implement the initContainer, including overcoming image pull and Security Context Constraint (SCC) challenges in OpenShift 4.19 with Red Hat Advanced Developer Suite (RHADS). Key RHADS components used:
- **Argo CD**: Deploys overlays from `clueforge-gitops` (auto-syncs on push, with UI for events/diffs).
- **Red Hat Developer Hub**: Verifies clones in UI Shell and topology graphs (Pod → InitContainer → Volume → PVC).
- **Konflux**: Implicit in main image builds (ready for custom git images with **Trusted Profile Analyzer (TPA)** scans and **Trusted Artifact Signer (TAS)** signing).
- **Trusted Profile Analyzer (TPA)**: Pre-push scans for SCC compliance (e.g., "UID in range?").
- **OpenShift Pipelines (Tekton)**: Teaser for migrating cloner to a scheduled Pipeline task.

## Prerequisites
- **PVC Setup**: `clueforge-repo-workspace` bound to ODF CephFS (10Gi RWX, mounted as `/opt/app-root/src/workspace`—see previous summary).
- **Namespace**: `tssc-app-development` (UID range dynamic, e.g., [1001020000, 1001029999]—check with `oc describe ns tssc-app-development`).
- **Repository Structure**: `clueforge-gitops` with `components/clueforge/overlays/development/` containing `kustomization.yaml` and `deployment-patch.yaml`.
- **Verification Tool**: **Red Hat Developer Hub** (Developer perspective > Topology > Pod > Shell) for console checks, or `oc exec` for CLI.
- **Security**: **TPA** scans pre-push (console > **Operators** > Installed Operators > Trusted Profile Analyzer > Scan tab > Git repo > Folder > Run "RHADS Tasks & Security").

## Verified Steps for Git InitContainer and Repo Cloning

### Step 1: Add InitContainer to Deployment Patch (Define Git Clone Logic)
Define the initContainer in the Deployment patch for Kustomize to merge with the base. Use a public image to avoid auth issues.

**File**: `components/clueforge/overlays/development/deployment-patch.yaml` (append under `spec.template.spec`)

```yaml
spec:
  template:
    spec:
      initContainers:
        - name: repo-cloner
          image: alpine/git:latest  # Public Docker Hub, no auth—pulls instantly
          securityContext:
            runAsUser: 1001020000  # In-range for restricted-v2 SCC (dynamic; check `oc describe ns tssc-app-development`)
            fsGroup: 1001020000  # Matches CephFS group for workspace permissions
          command: ['sh', '-c']
          args:
            - |
              cd /opt/app-root/src/workspace
              if [ ! -d "assisted-service" ]; then
                git clone https://github.com/openshift/assisted-service.git assisted-service
                cd assisted-service && git checkout master
              else
                cd assisted-service && git pull origin master
              fi
              echo "Assisted-service cloned/pulled to workspace"
          volumeMounts:
            - name: repo-workspace-volume
              mountPath: /opt/app-root/src/workspace
      # Your existing containers, volumes
```

**Deployment**:
- **TPA Scan**: Console > **Operators** > Installed Operators > Trusted Profile Analyzer > Scan tab > Git repo `clueforge-gitops` > Folder `overlays/development` > Run "RHADS Tasks & Security" (green for UID in range).
- Commit and push: `git add deployment-patch.yaml && git commit -m "Add assisted-service cloner initContainer with alpine/git" && git push origin main`.
- **Argo CD Sync**: Auto-triggers (or manual Sync Now in UI > Applications > `clueforge-development`).

**Verification**:
- Local: `cd components/clueforge && kustomize build overlays/development | grep -A10 "initContainers"` (renders cloner with UID 1001020000).
- **Argo CD UI**: Events: "Updated Deployment clueforge", "Pod rollout started".

### Step 2: Handle Image Pull Issues (Public Image Fallback)
Initial attempts with `registry.redhat.io/rhel9/git:latest` failed on auth ("unauthorized: Please login to the Red Hat Registry"). Quay.io/alpine/git:2.42 and bitnami/git:2.42 failed on manifest unknown. Switched to `alpine/git:latest` (public Docker Hub, valid manifest).

**Deployment**:
- Edit image in Step 1 patch, push, sync as above.

**Verification**:
- **Argo CD UI**: Events: "Pulling image 'alpine/git:latest' succeeded" (3.4s pull, 96MB).
- **Developer Hub**: Developer perspective > Topology > Pod > Events: "Started container repo-cloner", "Cloned to workspace".

### Step 3: Address SCC UID Range Mismatches (Dynamic Namespace Ranges)
SCC restricted-v2 enforced UID ranges (initial [1000930000, 1000939999], then [1000970000, 1000979999], latest [1001020000, 1001029999]). Updated `runAsUser` and `fsGroup` each time to match (from `oc describe ns tssc-app-development`).

**Deployment**:
- Edit securityContext in Step 1 patch to current range (e.g., 1001020000), push, sync.
- **TPA Scan**: Flags "UID in range?"—green after update.

**Verification**:
- **Argo CD UI**: Events: "SCC validation succeeded", no "forbidden: unable to validate".
- **Developer Hub**: Topology > Pod > Overview: Status "Running", no SCC error.

### Step 4: Resolve Argo Sync Drift and Immutable PVC (Prune/Replace)
Drift in securityContext.fsGroup (auto-injected by OpenShift for CephFS) caused OutOfSync. PVC immutable error ("spec is immutable after creation") during Replace.

**Deployment**:
- **Argo CD UI**: Sync Now > Check Prune + Replace + `--resource-exclusions=kind=PersistentVolumeClaim,name=clueforge-repo-workspace` (skips PVC re-apply) + `--timeout 300s`.
- Alternative: Comment out PVC in kustomization.yaml after initial bind.

**Verification**:
- **Argo CD UI**: Overview: "Synced" (green), Diff: No mismatches.
- **Developer Hub**: Topology: Pod → Volume → PVC (bound), Events: "Sync succeeded".

## Verification of Git Clone
- **Developer Hub Shell**: Developer perspective > Topology > Pod > Actions > Shell:
  - `ls /opt/app-root/src/workspace` (shows `assisted-service`).
  - `ls /opt/app-root/src/workspace/assisted-service` (git files: swagger.yaml).
  - `cd /opt/app-root/src/workspace/assisted-service && git log --oneline -1` (latest master commit).
  - `df -h /opt/app-root/src/workspace` (10Gi used ~100Mi).
- CLI: `oc exec -it <POD_NAME> -n tssc-app-development -- ls /opt/app-root/src/workspace/assisted-service` (POD_NAME from `oc get pods | grep clueforge`).
- Persistence: Delete pod (UI Actions) > New pod Shell > `ls /opt/app-root/src/workspace/assisted-service` (intact via CephFS).
- Logs: Developer Hub Logs tab > Filter "initContainer" (shows "Cloned to workspace").

## Troubleshooting
- **ImagePullBackOff**: Use public Docker Hub images (alpine/git:latest)—no auth. Check events for "unauthorized" (add pull secret for RH registry).
- **SCC Forbidden**: Update runAsUser/fsGroup to namespace range (`oc describe ns tssc-app-development`). **TPA** flags pre-push.
- **Sync Timeout/Drift**: Argo UI Sync > Prune + Replace + exclusions for PVC. **Developer Hub** scorecards alert on OutOfSync.
- **Clone Fails**: Logs show "fatal: could not read Username" (NetworkPolicy—allow github.com egress).

## Future Steps
- **Add More Repos**: Copy if-block in args for openshift-installer (main branch).
- **Port Full App**: Push code to `clueforge` repo > **Konflux** chain (Developer perspective > Builds > Create from component > TPA scan + TAS sign) > Patch for port 5000.
- **Automate Cloner**: **OpenShift Pipelines (Tekton)** > Pipeline from `app-of-apps/ci-tekton.yaml` (workspace-bound PVC for scheduled git pull).
- **Monitor**: **Red Hat Developer Hub** component scorecards for "Cloned? `ls /opt/app-root/src/workspace | wc -l >1`".

This setup enables ClueForge's git diffs resiliently, leveraging RHADS for declarative, secure deployments. Test a summary: Hub Shell `curl localhost:8081/api/generate_diffs -d '{"product":"assisted-service","repo":"assisted-service","file":"Swagger","last_n":1}'`.
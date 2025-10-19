# Setting Up Persistent Volume Claim (PVC) and File System for ClueForge in RHADS

ClueForge is a Python Flask application utilizing PyTorch for generating summaries of git diff changes in OpenShift repositories. To support its core functionality—cloning and storing git repositories for diff analysis and model-based summarization—a Persistent Volume Claim (PVC) and associated file system are required. This ensures repository data persists across pod restarts, deployments, and scaling events, enabling reliable git operations without data loss.

This guide outlines the verified steps to provision and mount the PVC using Red Hat Advanced Developer Suite (RHADS) components, including **Argo CD** for GitOps deployments, **Kustomize** for overlay management, and **Red Hat Developer Hub** for verification and observability. These steps assume a working Argo CD Application (`clueforge-development`) watching the `clueforge-gitops` repository in the `tssc-app-development` namespace, with OpenShift Data Foundation (ODF) configured for CephFS storage.

## Prerequisites
- **Namespace**: `tssc-app-development` (created via `CreateNamespace=true` in Argo spec).
- **StorageClass**: `ocs-storagecluster-cephfs` (ODF CephFS for shared file system semantics; fallback to `ocs-storagecluster-ceph-rbd` if unavailable).
- **Repository Structure**: `clueforge-gitops` with `components/clueforge/overlays/development/` folder containing `kustomization.yaml` and `deployment-patch.yaml`.
- **Verification Tool**: Use **Red Hat Developer Hub** (Developer perspective > Topology > Pod > Shell) for console-based checks, or `oc exec` for CLI validation.
- **Security**: Pre-push scans with **Trusted Profile Analyzer (TPA)** in console > **Scan** > Git repo (policy: "RHADS Storage" for PVC/mount risks).

## Verified Steps for PVC and File System Setup

### Step 1: Create the PVC YAML File
Define the PVC to provision 10Gi of CephFS storage for git repositories. This resource is stored in the development overlay for environment-specific control.

**File**: `components/clueforge/overlays/development/pvc-repo-workspace.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: clueforge-repo-workspace
spec:
  accessModes:
    - ReadWriteOnce  # RWX for multi-pod scaling if needed later
  resources:
    requests:
      storage: 10Gi
  storageClassName: ocs-storagecluster-cephfs  # ODF CephFS for POSIX file system
```

**Deployment**:
- Commit and push: `git add pvc-repo-workspace.yaml && git commit -m "Add PVC for git repo workspace" && git push origin main`.
- **Argo CD Sync**: Auto-triggers (or manual Sync Now in UI > Applications > `clueforge-development`).

**Verification**:
- CLI: `oc get pvc clueforge-repo-workspace -n tssc-app-development` (Status: Bound, Capacity: 10Gi).
- Console: Developer Hub Topology > PVC icon > YAML (bound to ODF PV).
- **RHADS Tie-In**: **TPA** scan pre-push flags issues like "Access modes secure?".

### Step 2: Include the PVC in Kustomization.yaml
Kustomize aggregates resources for Argo CD to deploy. Add the PVC to `resources:` to ensure it's included in the overlay build.

**File**: `components/clueforge/overlays/development/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: tssc-app-development  # Injects namespace to all resources
resources:
  - ../../base  # Base Deployment, Service, Route
  - pvc-repo-workspace.yaml  # Include PVC for storage provisioning
patchesStrategicMerge:
  - deployment-patch.yaml  # Existing patches (replicas, image)
```

**Deployment**:
- Commit and push: `git add kustomization.yaml && git commit -m "Include PVC in dev kustomization" && git push origin main`.
- **Argo CD Sync**: Auto-syncs; UI Events show "Applied PersistentVolumeClaim clueforge-repo-workspace".

**Verification**:
- Local: `cd components/clueforge && kustomize build overlays/development | grep -A5 "kind: PersistentVolumeClaim"` (renders PVC with namespace).
- Console: Developer Hub Topology > New PVC edge in graph (PVC → ODF).
- **RHADS Tie-In**: **Argo CD** Diff tab confirms repo vs. cluster match.

### Step 3: Mount the Volume in Deployment Patch
Wire the PVC to the pod by defining a volume and mounting it to the container's workspace path (`/opt/app-root/src/workspace` for UBI/Flask compatibility). This makes the CephFS file system available inside the container.

**File**: `components/clueforge/overlays/development/deployment-patch.yaml` (append to existing content, e.g., replicas/image)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clueforge
spec:
  replicas: 1  # Existing
  template:
    spec:
      containers:
        - name: container-image  # Existing scaffold container
          image: quay.io/rhads-jowilkin/clueforge:2f56f161179fe270f71ad5e842df714934f52a47@sha256:0994d503fe4d2efc761d042fb4497ab57a3cf56c40deff97fbbb38f9324255e6  # Existing
          volumeMounts:  # Mount PVC as file system
            - name: repo-workspace-volume
              mountPath: /opt/app-root/src/workspace  # Container path for git repos
          # Existing ports, probes, resources
      volumes:  # Define volume from PVC
        - name: repo-workspace-volume
          persistentVolumeClaim:
            claimName: clueforge-repo-workspace
      securityContext:  # Optional: CephFS permissions
        fsGroup: 1001  # Non-root group for ODF (matches exec output group 1000930000)
```

**Deployment**:
- Commit and push: `git add deployment-patch.yaml && git commit -m "Mount PVC as workspace volume in dev Deployment" && git push origin main`.
- **Argo CD Sync**: Rolling update; UI Events: "Updated Deployment clueforge", pod rollout.

**Verification**:
- CLI: `oc exec -it <POD_NAME> -n tssc-app-development -- ls -ld /opt/app-root/src/workspace` (drwxrwsr-x dir, owned by root/1000930000).
- CLI: `oc exec ... -- touch /opt/app-root/src/workspace/test_file.txt && ls -l /opt/app-root/src/workspace/` (file created/listed).
- CLI: `oc exec ... -- rm /opt/app-root/src/workspace/test_file.txt` (removed; persists across pod delete/recreate).
- Console: Developer Hub Topology > Pod > Shell > `df -h /opt/app-root/src/workspace` (10Gi CephFS), `mount | grep workspace` (CephFS type rw).
- **RHADS Tie-In**: **Developer Hub** graph shows Pod → Volume → PVC; **TPA** pre-push validates "mount path secure?".

## Future Steps: Cloning Repositories into the Workspace
Once the mount is verified, the next phase is populating `/opt/app-root/src/workspace` with git repositories (e.g., assisted-service). This will be added as an initContainer in the Deployment patch for idempotent cloning/pulls, deployed via Argo CD. Verified steps will be appended here after testing, including **OpenShift Pipelines (Tekton)** for scheduled automation and **Konflux** for image rebuilds with updated paths (e.g., REPO_BASE env var).

For troubleshooting:
- **Argo CD UI**: Applications > `clueforge-development` > Diff/Events for sync issues.
- **Developer Hub**: Topology for graphs, Shell for checks, Scorecards for automated health (e.g., "Workspace Mounted? `df -h /opt/app-root/src/workspace | grep 10Gi`").

This setup ensures ClueForge's git operations are resilient, leveraging RHADS for declarative, secure, and observable deployments.
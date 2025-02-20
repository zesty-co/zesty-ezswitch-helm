# EZSwitch - Zesty StatefulSet Migration Tool

**EZSwitch** is a Kubernetes utility designed to simplify the migration of stateful workloads from non-scalable PVCs to Zesty Disk PVCs with auto-scaling capabilities. The tool manages the entire lifecycle of this storage transition—from initial synchronization to final switchover—while offering flexibility in how migrations are initiated and controlled.

**Table of Contents**  
- [Overview](#overview)  
- [Prerequisites](#prerequisites)  
- [Installation](#installation)
  - [Kubectl-Zesty Plugin (Quickstart)](#kubectl-zesty-plugin-quickstart)  
  - [Manual Installation](#manual-installation)
- [The EZSwitch Custom Resource](#the-ezswitch-custom-resource)  
- [Migration Phases (.status.status)](#migration-phases-statusstatus)  
- [Commands](#commands)  
  - [start](#start-stsname)
  - [migrate](#migrate-stsname)
  - [pause](#pause-stsname)
  - [resume](#resume-stsname)
  - [abort](#abort-stsname)
  - [rollback](#rollback-stsname)
  - [status](#status-stsname)
  - [set](#set-stsname)
- [Cleanup](#cleanup)  
- [Logging and Monitoring](#logging-and-monitoring)  
- [GitOps and Version Control](#gitops-and-version-control) 
- [Troubleshooting](#troubleshooting)  
- [Limitations and Considerations](#limitations-and-considerations)  
- [References](#references)  

---

## Overview

EZSwitch's primary purpose is to leverage Zesty's auto-scaling storage solution ([Kompass Storge](https://docs.zesty.co/docs/storage-optimization)). By migrating existing StatefulSets (STS) to Kompass Storage PersistentVolumeClaims (PVCs), your workloads gain the ability to automatically scale storage capacity as data usage grows or shrinks, eliminating the need for manual PVC resizing.

EZSwitch operates as a Kubernetes Custom Resource (CR) managed by a controller. It coordinates syncing jobs (using [rsync](https://github.com/RsyncProject/rsync) under the hood), the final switchover to a Zesty-based StatefulSet (with the same properties as the initial STS), and supports both automatic or manual migration phases.

---

## Prerequisites

Before installing EZSwitch, ensure you have:

1. **Kompass Storage**: Install and configure the [zesty-helm](https://github.com/zesty-co/zesty-helm) package in your cluster
2. **Kubernetes**: Version 1.24 or higher
3. **Permissions**: Cluster-admin access to deploy the controller and CRDs
4. **Namespace**: The system will use the `zesty-ezswitch` namespace by default

## Installation

Kompass Storage EZSwitch can be installed and operated in two ways:
1. Automaticlly using the kubectl-zesty plugin (recommended for most users)
2. Manual (recommended for GitOps workflows)

### Kubectl-Zesty Plugin (Quickstart)
The kubectl-zesty plugin provides a streamlined command-line interface for EZSwitch operations. When using the plugin, it will:
1. Install the `zesty-ezswitch-helm` chart automatically
2. Create and manage the required [EZSwitch custom resources](#the-ezswitch-custom-resource)
3. Provide easy access to [migration status](#status) and controls

Before starting, install the kubectl-zesty plugin:
[kubectl-zesty plugin installation guide](https://github.com/zesty-co/kubectl-plugin)

To begin a migration:
```bash
kubectl zesty ezswitch start <stsName> [--autoMigrate=<true|false>] [--helm-namespace=<namespace>] [--set key=value ...]
```

Example:
```bash
kubectl zesty ezswitch start myapp-sts --autoMigrate=false --set logLevel=4
```

> **Note**: EZSwitch resources are installed in the `zesty-ezswitch` namespace by default. This can be overridden using the `--helm-namespace` option, as detailed in [commands](#commands).

### Manual Installation

For users who prefer direct Helm management (such as GitOps workflows) or don't want to use the kubectl-zesty plugin, follow these steps:

#### 1. Add the Zesty EZSwitch repository to Helm.
```bash
# Add the Helm repository
helm repo add zestyezswitchrepo https://zesty-co.github.io/zesty-ezswitch-helm
# Install EZSwitch
helm install zesty-ezswitch [-n <NAMESPACE>] zestyezswitchrepo/ezswitch
```
> **Note** Additional Helm settings can be found in the section [#helm-chart-values](#helm-chart-values)

#### 2. Create EZSwitch CR
Create a YAML file (for example, `ezswitch-resource.yaml`) that includes your EZSwitch configurations as outlined in [The EZSwitch Custom Resource](#the-ezswitch-custom-resource)

Example:
```yaml
apiVersion: storage.zesty.co/v1alpha1
kind: EZSwitch 
metadata:
  name
  namespace: default # Namespace should match stsNamespace
spec:
  stsName: myapp-sts # Name of source statefulset to be migrated
  stsNamespace: default # Namespace of source statefulset
  autoMigrate: true # Sets migration process to migrate as soon as all data is copied
  zestyStsName: myapp-sts-zesty # Name of target statefulset to migrate into
```
> **Note**: Setting `autoMigrate: true` will run the migration automatically from start to finish. Setting `autoMigrate: false` allows you to manually control when to proceed between migration phases. See full CR options in the section [#the-ezswitch-custom-resource](#the-ezswitch-custom-resource)

#### 3. Apply the EZSwitch resource:
The migration will start running once the EZswitch CR is applied
```bash
kubectl apply -f ezswitch-resource.yaml
```

> **Note**: Deleting the EZSwitch resource during an ongoing migration will trigger a rollback, undoing all changes made so far. For the rollback to function correctly, the original StatefulSet must still be present.

#### 4. Cleanup
1. **Abort EZSwitch**
```bash
kubectl delete ezswitch <ezswitch-name>
```
2. **Uninstall Helm**:
```bash
helm uninstall zesty-ezswitch [--helm-namespace <NAMESPACE>]
```

---

## Helm Chart Values
| Key                | Default                                                                               | Description                                             |
|--------------------|---------------------------------------------------------------------------------------|---------------------------------------------------------|
| logLevel           | 6                                                                                     | Log level for the ezswitch-controller.                 |
| controller.image   | zd/k8s/ezswitch-controller                                                            | The controller container image.                         |
| controller.tag     | latest                                                                                | The controller image tag.                               |
| syncJob.image      | zd/k8s/sync-pvcs                                                                      | The sync job container image (for data migration).      |
| syncJob.tag        | latest                                                                                | The sync job image tag.                                 |

> **Note**: The `logLevel` sets the verbosity level based on the [klog](https://github.com/kubernetes/klog) style. Higher numbers mean more verbose logs.

---

## The EZSwitch Custom Resource

An `EZSwitch` resource specifies the intended migration actions. Important fields are:

- **`.spec.stsName`**: Name of the original StatefulSet.
- **`.spec.stsNamespace`**: Namespace of the original StatefulSet.
- **`.spec.autoMigrate`**:  
  - `true` (default): Executes a fully automated process (sync, scale down the old STS, final sync, deploy the new STS).
  - `false`: Halts at `Syncing`, maintaining the original STS and continuously syncing PVCs until manually resumed.
- **`.spec.zestyStsName`**: Name of the new Zesty-backed StatefulSet (defaults to `<stsName>-zesty`).

If not using the [zesty-plugin](#kubectl-zesty-plugin-quickstart), you can manage migration behavior with these fields:

- **`.status.phase`**: 
  - `Pausing` - Pauses ezswitch migration
  - `Activating` - Resumes ezswitch migration
- **`.spec.transferRateLimits`**: Specifies the maximum transfer rate for sync jobs in kb/s (utilizes rsync's bwlimit argument).
- **`.spec.autoMigrate`**: Initially determines if the migration process is fully automated or semi-controlled. For semi-controlled migration (`autoMigrate=false`), change `.spec.autoMigrate` to `true` to continue the migration process.


---

## Migration Statuses (.status.status)

1. **InstallRequirements**: Sets up necessary RBAC permissions and prerequisites.  
2. **Init**: Creates corresponding PVCs under the Zesty storage class.  
3. **CreateSyncJobs**: Launches sync jobs to copy data from the original PVCs to the new PVCs.  
4. **Syncing**: Continuously synchronizes data to ensure the new PVCs mirror the old ones in near real-time.  
5. **ReadyForMigration**: The migration halts here if `autoMigrate=false`. If `autoMigrate=true`, it proceeds by scaling down the original STS.  
6. **CreatingFinalSyncJobs**: Sets up final sync jobs to ensure data consistency before the final cutover.  
7. **SyncingFinalJobs**: Runs a final round of synchronization to confirm all data is up to date.  
8. **WaitingForZestySTS**: Deploys the new Zesty-based STS and waits for it to become fully ready.  
9. **Success**: Cleans up finalizers and completes the migration.

> **Note**: If `autoMigrate=false`, the process pauses at `Syncing` until resumed with the `migrate` command or by setting `autoMigrate=true` on the `EZSwitch` resource.

---

## Commands

All EZSwitch commands are invoked via the CLI plugin:
```bash
kubectl zesty ezswitch <command> <stsName> [flags]
```
Below are the available commands, their flags, and descriptions.

### start \<stsName\>

Starts the migration process, installing EZSwitch components (including the Helm chart if not already installed) and creating the EZSwitch CR. By default all ezswitch resources will be on the `zesty-ezswitch` namespace

| Flag               | Default           | Description                                                            |
|--------------------|-------------------|------------------------------------------------------------------------|
| --autoMigrate      | true              | Run process automatically to completion if true.                        |
| --stsNamespace     | default           | Namespace of the original STS.                                          |
| --set key=value    | (none)            | Override [Helm chart values](#helm-chart-values) during installation.   |
| --logLevel int     | 4                 | Set the log level of ezswitch-controller.                               |
| --zestyStsName     | \<stsName\>-zesty | Name of the new Zesty-based STS.                                        |
| --helm-namespace   | zesty-ezswitch    | Namespace where ezswitch resources will be created                      |

**Example**:
```bash
kubectl zesty ezswitch start myapp-sts --autoMigrate=false --helm-namespace=custom-namespace --set logLevel=4
```
### migrate \<stsName\>

Resumes the migration if previously halted at `ReadyForMigration`.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch migrate myapp-sts
```
### pause \<stsName\>

Pauses the ongoing migration.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch pause myapp-sts
```
### resume \<stsName\>

Resumes a previously paused migration.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch resume myapp-sts
```
### abort \<stsName\>

Aborts the current process before significantly impacting the original STS and cleans up created resources.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch abort myapp-sts
```
### rollback \<stsName\>

Attempts to revert to the original state if changes to the STS have begun.

> **Note**:  
> - The `EZSwitch` resource from which the migration started must still exist.  
> - The original STS and its PVCs must still exist.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch rollback myapp-sts
```
### status \<stsName\>

Displays the current migration status, including phase and sync progress.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch status myapp-sts
```
### set \<stsName\>

Modifies attributes of the `EZSwitch` CR.

| Flag                        | Default | Description                                                |
|-----------------------------|---------|------------------------------------------------------------|
| --autoMigrate=<bool>        | true    | Updates autoMigrate field.                                 |
| --transferRateLimits=<int>  | -1      | Updates transferRateLimits field (-1 will unset its value).|

**Example**:
```bash
# Disable automatic migration
kubectl zesty ezswitch set myapp-sts --autoMigrate=false
```
---

### cleanup \<stsName\>

Removes the `EZSwitch` CR.

| Flag              | Default        | Description                                                           |
|-------------------|----------------|-----------------------------------------------------------------------|
| --helm-namespace  | zesty-ezswitch | Namespace of the original STS.                                        |
| --delete-old-sts  | false          | Deletes the old StatefulSet on cleanup.                               |
| --force-abort     | false          | Forces abort of all EZSwitch resources before cleanup.                |
| --keep-resources  | false          | Keeps EZSwitch resources instead of deleting them.                    |
| --stsNamespace    | default        | Namespace of the original STS.                                        |

**Example**:
```bash
kubectl zesty ezswitch cleanup <stsName> [flags]
```

---

## Logging and Monitoring

- The EZSwitch controller (`zesty-ezswitch-controller`) runs in the `ezswitch` namespace.
- View logs: `kubectl logs deploy/zesty-ezswitch-controller -n ezswitch`
- Check EZSwitch events and status:
```bash
kubectl describe ezswitch myapp-sts
```
> **Note**: Unlike Kompass Storage, there is no dedicated Prometheus exporter for EZSwitch.

---

## GitOps and Version Control

For GitOps workflows (e.g., Argo CD), commit `EZSwitch` resources to a Git repository. Start with `autoMigrate=false` and later push a commit changing it to `true` to trigger the final switchover. This ensures changes are tracked, reviewed, and rolled out through a GitOps process.

---

## Troubleshooting
- If issues occur, use `abort` or `rollback` to halt or revert the process.
- Check the [controller logs](#logging-and-monitoring)
- Ensure the [zesty-helm](https://github.com/zesty-co/zesty-helm) chart is correctly installed.
- Contact Zesty Support for advanced troubleshooting.

---

## Limitations and Considerations

- Only one `ezswitch` migration per STS is currently supported.
- Ensure sufficient cluster resources for sync jobs (CPU, memory).
- Avoid scaling the STS during migration.
- Migration duration depends on PVC size and cluster load.

---

## References

- **Zesty Official Documentation**:
  - [Kompass Storge](https://docs.zesty.co/docs/storage-optimization)
  - [Deploy Kompass Storge for Kubernetes](https://docs.zesty.co/docs/deploy-pvs-autoscaling)
- **Helm Repositories**:
  - [zesty-helm (GitHub)](https://github.com/zesty-co/zesty-helm)
- **CLI Plugin**:
  - [kubectl-zesty plugin (GitHub)](https://github.com/zesty-co/kubectl-plugin)

EZSwitch streamlines the path to auto-scaling storage with Kompass Storge. By following these commands and guidelines—whether via the CLI or Helm—you can seamlessly manage the entire migration lifecycle. Enjoy the benefits of automatic storage scaling with minimal manual overhead!
# EZSwitch - Zesty StatefulSet Migration Tool

**EZSwitch** is a Kubernetes utility designed to simplify the migration of stateful workloads from non-scalable PVCs to Zesty Disk PVCs with auto-scaling capabilities. The tool manages the entire lifecycle of this storage transition—from initial synchronization to final switchover—while offering flexibility in how migrations are initiated and controlled.

**Table of Contents**  
- [Overview](#overview)  
- [Prerequisites](#prerequisites)  
- [Quickstart](#quickstart)  
  - [Kubectl-Zesty Plugin (Quickstart)](#kubectl-zesty-plugin-quickstart)  
  - [Manual Installation](#manual-installation)
    - [Add the Zesty repository to Helm](#add-the-zesty-ezswitch-repository-to-helm)
    - [Update a configured repository](#update-a-configured-repository)
    - [Install the chart](#install-the-chart)
    - [Helm installation flags](#helm-installation-flags)
    - [Uninstalling the chart](#uninstalling-the-chart)
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
- [Logging and Monitoring](#logging-and-monitoring)  
- [GitOps and Version Control](#gitops-and-version-control)  
- [Troubleshooting](#troubleshooting)  
- [Limitations and Considerations](#limitations-and-considerations)  
- [Cleanup](#cleanup)  
- [References](#references)  

---

## Overview

EZSwitch’s primary purpose is to leverage Zesty’s auto-scaling storage solution ([zesty-disk](https://docs.zesty.co/docs/zesty-disk)). By migrating existing StatefulSets (STS) to Zesty Disk PersistentVolumeClaims (PVCs), your workloads gain the ability to automatically scale storage capacity as data usage grows or shrinks, eliminating the need for manual PVC resizing.

EZSwitch operates as a Kubernetes Custom Resource (CR) managed by a controller. It coordinates syncing jobs (using [rsync](https://github.com/RsyncProject/rsync) under the hood), the final switchover to a Zesty-based StatefulSet (with the same properties as the initial STS), and supports both automatic or manual migration phases.

---

## Prerequisites
- **Zesty Helm Chart**: The [zesty-helm](https://github.com/zesty-co/zesty-helm) package must be installed and configured in your cluster. This provides the underlying Zesty Disk functionality that EZSwitch requires.
- **Kubernetes Version**: Requires Kubernetes v1.24 or higher.
- **Permissions**: Must have cluster-admin permissions to deploy and manage the EZSwitch controller and related CRDs.
- **Namespace**: By default, EZSwitch and related components run in the `ezswitch` namespace.
---

## Installation

EZSwitch can be set up in two main ways:

### Kubectl Zesty Plugin (Quickstart)
This is best for quick, streamlined migrations via a set of helpful commands. These commands will install zesty-ezswitch-helm chart, create and edit ezswitch CRDs, report the migration and more. See the [commands](#commands) section for more info.

Make sure kubectl zesty plugin is installed and up to date by following the kubectl zesty plugin installation steps: [kubectl-zesty plugin](https://github.com/zesty-co/kubectl-plugin)

```bash
kubectl zesty ezswitch start <stsName> [--autoMigrate=<true|false>] [--set key=value ...]
```
For example:
```bash
kubectl zesty ezswitch start myapp-sts --autoMigrate=false --set logLevel=4
```


### Manual Installation

If you’d rather manage the Helm release directly (e.g., via GitOps, or you don’t wish to install the kubectl-zesty plugin), you can install the chart yourself:

#### Add the Zesty Ezswitch repository to Helm
```bash
helm repo add zestyezswitchrepo https://zesty-co.github.io/zesty-ezswitch-helm
```
#### Update a configured repository
```bash
helm repo update
```
#### Install the chart
```bash
helm install zesty-ezswitch \
  [-n <NAMESPACE>] \
  zestyezswitchrepo/zesty-ezswitch-helm
```

#### Helm installation flags
| Key                | Default                                                                               | Description                                             |
|--------------------|---------------------------------------------------------------------------------------|---------------------------------------------------------|
| logLevel           | 6                                                                                      | Log level for the ezswitch-controller.                 |
| controller.image   | 672188301118.dkr.ecr.eu-west-1.amazonaws.com/zd/k8s/ezswitch-controller               | The controller container image.                         |
| controller.tag     | latest                                                                                | The controller image tag.                               |
| syncJob.image      | 672188301118.dkr.ecr.eu-west-1.amazonaws.com/zd/k8s/sync-pvcs                         | The sync job container image (for data migration).      |
| syncJob.tag        | latest                                                                                | The sync job image tag.                                 |


> **Note**: The `logLevel` sets the verbosity level based on the [klog](https://github.com/kubernetes/klog) style. Higher numbers mean more verbose logs.

#### Uninstalling the chart
```bash
helm delete zesty-ezswitch [-n <NAMESPACE>]
```
---

## The EZSwitch Custom Resource

An `EZSwitch` resource defines the desired migration behavior. Key fields include:

- **`.spec.stsName`**: The original StatefulSet name.
- **`.spec.stsNamespace`**: The original StatefulSet namespace.
- **`.spec.autoMigrate`**:  
  - `true` (default): Fully automated process (sync, scale-down old STS, final sync, deploy new STS).
  - `false`: Stops at `ReadyForMigration`, keeping the original STS running and PVCs continuously synced until you manually resume.
- **`.spec.zestyStsName`**: The name of the new Zesty-backed StatefulSet (defaults to `<stsName>-zesty`).

**Example:**
```yaml
apiVersion: storage.zesty.co/v1alpha1
kind: EZSwitch
metadata:
  name: myapp-sts-ezswitch
  namespace: default
spec:
  stsName: myapp-sts
  stsNamespace: default
  autoMigrate: true
  zestyStsName: myapp-sts-zesty
```
To create it:
```bash
kubectl apply -f ezswitch-resource.yaml
```
---

## Migration Phases (.status.status)

1. **InstallRequirements**: Sets up necessary RBAC permissions and prerequisites.  
2. **Init**: Creates corresponding PVCs under the Zesty storage class.  
3. **CreateSyncJobs**: Launches sync jobs to copy data from the original PVCs to the new PVCs.  
4. **Syncing**: Continuously synchronizes data to ensure the new PVCs mirror the old ones in near real-time.  
5. **ReadyForMigration**: The migration halts here if `autoMigrate=false`. If `autoMigrate=true`, it proceeds by scaling down the original STS.  
6. **CreatingFinalSyncJobs**: Sets up final sync jobs to ensure data consistency before the final cutover.  
7. **SyncingFinalJobs**: Runs a final round of synchronization to confirm all data is up to date.  
8. **WaitingForZestySTS**: Deploys the new Zesty-based STS and waits for it to become fully ready.  
9. **Success**: Cleans up finalizers and completes the migration.

> **Note**: If `autoMigrate=false`, the process pauses at `ReadyForMigration` until resumed with the `migrate` command or by setting `autoMigrate=true` on the `EZSwitch` resource.

---

## Commands

All EZSwitch commands are invoked via the CLI plugin:
```bash
kubectl zesty ezswitch <command> <stsName> [flags]
```
Below are the available commands, their flags, and descriptions.

### start <stsName>

Starts the migration process, installing EZSwitch components (including the Helm chart if not already installed via the CLI approach) and creating the EZSwitch CR.

| Flag             | Default           | Description                                                            |
|------------------|-------------------|------------------------------------------------------------------------|
| --autoMigrate    | true             | Run process automatically to completion if true.                       |
| --stsNamespace   | default          | Namespace of the original STS.                                         |
| --set key=value  | (none)           | Override Helm chart values during installation.                        |
| --logLevel int   | 4                | Set the log level of ezswitch-controller.                              |
| --zestyStsName   | <stsName>-zesty  | Name of the new Zesty-based STS.                                       |

**Example**:
```bash
kubectl zesty ezswitch start myapp-sts --autoMigrate=false --set logLevel=4
```
### migrate <stsName>

Resumes the migration if previously halted at `ReadyForMigration`.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch migrate myapp-sts
```
### pause <stsName>

Pauses the ongoing migration.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch pause myapp-sts
```
### resume <stsName>

Resumes a previously paused migration.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch resume myapp-sts
```
### abort <stsName>

Aborts the current process before significantly impacting the original STS and cleans up created resources.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch abort myapp-sts
```
### rollback <stsName>

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
### status <stsName>

Displays the current migration status, including phase and sync progress.

| Flag           | Default   | Description                        |
|----------------|-----------|------------------------------------|
| --stsNamespace | default   | Namespace of the original STS.     |

**Example**:
```bash
kubectl zesty ezswitch status myapp-sts
```
### set <stsName>

Modifies attributes of the `EZSwitch` CR.

| Flag                 | Default | Description                       |
|----------------------|---------|-----------------------------------|
| --autoMigrate=<bool> | true    | Updates autoMigrate field.        |

**Examples**:
```bash
# Disable automatic migration
kubectl zesty ezswitch set myapp-sts --autoMigrate=false
```
---

## Logging and Monitoring

- The EZSwitch controller (`zesty-ezswitch-controller`) runs in the `ezswitch` namespace.
- View logs: `kubectl logs deploy/zesty-ezswitch-controller -n ezswitch`
- Check EZSwitch events and status:
```bash
kubectl describe ezswitch myapp-sts
```
> **Note**: Unlike Zesty Disk, there is no dedicated Prometheus exporter for EZSwitch.

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

## Cleanup

When the migration completes or if you choose not to proceed, you can remove EZSwitch resources in two ways:

1. **Helm Uninstall (if you installed manually)**:
```bash
helm delete zesty-ezswitch [-n <NAMESPACE>]
```
2. **Using the EZSwitch CLI**:
```bash
kubectl zesty ezswitch cleanup <stsName> [flags]
```
| Flag              | Default | Description                                                           |
|-------------------|---------|-----------------------------------------------------------------------|
| --delete-old-sts  | false   | Deletes the old StatefulSet on cleanup.                               |
| --force-abort     | false   | Forces abort of all EZSwitch resources before cleanup.                |
| --keep-resources  | false   | Keeps EZSwitch resources instead of deleting them.                    |
| --stsNamespace    | default | Namespace of the original STS.                                        |

---

## References

- **Zesty Official Documentation**:
  - [Zesty Disk](https://docs.zesty.co/docs/zesty-disk)
  - [Deploy Zesty Disk for Kubernetes](https://docs.zesty.co/docs/deploy-zesty-disk-for-k8s)
- **Helm Repositories**:
  - [zesty-helm (GitHub)](https://github.com/zesty-co/zesty-helm)
- **CLI Plugin**:
  - [kubectl-zesty plugin (GitHub)](https://github.com/zesty-co/kubectl-plugin)

EZSwitch streamlines the path to auto-scaling storage with Zesty Disk. By following these commands and guidelines—whether via the CLI or Helm—you can seamlessly manage the entire migration lifecycle. Enjoy the benefits of automatic storage scaling with minimal manual overhead!
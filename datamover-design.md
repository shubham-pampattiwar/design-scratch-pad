# Add Datamover to Velero
## Abstract
As of today, Velero supports CSI snapshots. But there is a missing gap in functionality, the CSI snapshots are not durable as well as portable.
It is desired that Velero support a concept of Datamover. We would like to propose an extensible design to support Datamovers, where-in the users
can bring their own datamover implementation and use it to replicate CSI snapshots to object storage during backup and also recreate PVCs from the snapshots
in object storage during restore. Thus making the CSI snapshots durable as well as portable.

## Background
Currently, Velero has two option to support backing up of persistent volumes.

1. Restic: This solution performs filesystem level backups and mostly used when a volume snapshot plugin is not available. However, the restic integration is 
tightly coupled with Velero, makes it difficult for someone to bring their own implementation of data movement. Although restic is portable but it is not 
crash consistent as it requires quiescing of data before the backup is taken thus affecting the availability of application.

2. Volume Snapshots: Storage providers provide with volume snapshot capabilities via plugins. These snapshots are crash consistent but there is no guarantee that
these snapshots are durable. These snapshots are generally stored locally in the cluster storage, so if the cluster breaks then the snapshots are of no use. Velero also supports
CSI volume snapshots, but as there is no data movement provided by Velero for these CSI snapshots they are not durable and dependent on the storage provider.


Related Issue: https://github.com/vmware-tanzu/velero/issues/3229

## Goals
- To be able to backup CSI snapshots to object storage
- To be able to recreate CSI snapshots under a different storage provider
- To provide an extensible data mover solution which should allow third parties to plugin their own datamover implementation
- To provide a Datamover example PoC

## Non Goals
- Maintain 3rd party data mover implementations
- Adding a status watch controller to Velero
- To support incremental backups (future scope, dependent on standard CSI support for snapshot differentials)
- To allow plug-ins for services like encryption, compression etc. (future scope)

## High-Level Design ([VolSync](https://github.com/backube/volsync) as a working Proof of Concept)
The goal of this high level design is to explain a working PoC of moving CSI snapshots where-in a datamover (VolSync) works in conjunction with Velero via a DataMover plugin (will be installed when the DataMover flag is set to true).

The following design proposal is for adding a datamover feature to Velero which can be integrated with various user implemented datamovers.

![](https://i.imgur.com/IvncXhE.png)

The design components and CRs that we are introducing are as follows:

1. DataMoverType(DMT): This is a cluster scoped Custom Resource that will have details about the data mover provisioner.The provisioner will be responsible
for creating a DataMover plugin that will create a `DataMoverBackup` CR & `DataMoverRestore` CR. They will also be responsible for implementing the logic
for DataMoverBackup & DataMoverRestore controller that corresponds to their data mover. The `DataMoverType` name will be referenced in `DataMoverBackup`
& `DataMoverRestore` CRs. This will help in selecting the data mover implementation during runtime. If the `DataMoverType` name is not defined, then the
default `DataMoverType` will be used, which in this case will be `VolSync`

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.7.0
  name: datamovertype.velero.io
spec:
  provisioner: <VolSync>
```

2. DataMoverBackup(DMB): Assuming that the flag to enable DataMover is set to true during Velero install, when a velero backup is created, it triggers DataMover plugin
to create the `DataMoverBackup` (DMB) CR in the app namespace. `DataMoverBackup` CR supports either a volume snapshot or a pvc as the type of the backup object. If the 
velero CSI plugin is used for backup, `VolumeSnapshot` is used as the type or else `PVC` is used.

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.7.0
  name: datamoverbackup.velero.io
spec:
  dataMoverType: <datamovertype name> 
  - type: <VolumeSnapshot|PVC>
    sourceClaimRef:
      name: <snapshot_content_name>|<pvc_name>
      namespace: <namespace>  //optional 
    config:     //optional based on the datamover impl

```

3. DataMoverRestore(DMR): When a velero restore is triggered, the DataMover plugin looks for `DataMoverBackup` in the backup resources. If it encounters a `DataMoverBackup` resource,
then the plugin will create a `DataMoverRestore` CR in the app namespace. It will populate the CR with the details obtained from the `DataMoverBackup` resource.

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.7.0
  name: datamoverrestore.velero.io
spec:
  dataMoverType: <datamovertype name>
  destinationClaimRef:
    name: <PVC_claim_name>
    namespace: <namespace>
  config:     //optional based on the datamover impl
```

Config section in the above CR is optional. It lets the user specify extra parameters needed by the data mover. For eg: VolSync data mover needs restic secret to perform backup & restore

eg:

```
apiVersion: velero.io/v1
kind: DataMoverRestore
metadata:
  name: volsync-datamover
spec:
  dataMoverType: volsync
  destinationClaimRef:
    name: <PVC_claim_name>
    namespace: <namespace>
  config:     //optional based on the datamover impl
    resticSecret:
      name: <secret_name>
```

4. DataMover Flag: This will be a Velero install flag, if this is flag is set to true then Velero will be installed with the DataMover Plugin.

5. DataMover Plugin: This is the plugin which will get installed when DataMover Flag is set to true. This plugin's duty would be to create DataMoverBackup(DMB) CR. This 
plugin would lookup for the PVCs present in the user namespace and will create a DMB CR for every PVC. This Plugin is also responsible for creating DataMoverRestore(DMR) CR for each DMB CR encountered during restore.

6. DataMoverBackup Controller: The DataMoverBackup Controller will watch for DataMoverBackup CR.

7. DataMoverRestore Controller: DataMoverRestore Controller will watch for DataMoverRestore CR.

**Note**: Currently, we have taken the DataMoverRestore Controller approach but we want to explore upstream K8s APIs for these usecases like [volume populator](https://kubernetes.io/blog/2021/08/30/volume-populators-redesigned/).

### Data Mover controller PoC - [VolSync](https://github.com/backube/volsync)

VolSync will be used as the Data Mover and `restic` (VolSync restic option and not the one integrated with Velero) will be the supported method for backup & restore of PVCs. Restic repository details are configured in a `secret` object
which gets used by the VolSync's resources. This design takes advantage of VolSync's two resources - `ReplicationSource` & `ReplicationDestination`. `ReplicationSource` object helps
with taking a backup of the PVCs and using restic to move it to the storage specified in the restic secret. `ReplicationDestination` object takes care of restoring the backup from the
restic repository. There will be a 1:1 relationship between the replication src/dest CRs and PVCs.

Here, the user will create a restic secret. Using that secret as source, the controller will create on-demand secrets for every backup/restore request.

Assuming that the DataMover flag is enabled, then the user needs to create a restic secret with all the following details (this should be done before velero install just like `cloud-credentials`),
```
apiVersion: v1
kind: Secret
metadata:
  name: restic-config
type: Opaque
stringData:
  # The repository url
  RESTIC_REPOSITORY: s3:s3.amazonaws.com/<bucket>
  # The repository encryption key
  RESTIC_PASSWORD: <password>
  # ENV vars specific to the chosen back end
  # https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html
  AWS_ACCESS_KEY_ID: <access_id>
  AWS_SECRET_ACCESS_KEY: <access_key>
```
*Note: More details for installing restic secret in [here](https://volsync.readthedocs.io/en/stable/usage/restic/index.html#specifying-a-repository)*


Like any other plugins, DataMover plugin gets deployed as an init container in the velero deployment. This plugin will be responsible for creating `DataMoverBackup` & `DataMoverRestore` CRs.

Once a DataMoverBackup CR gets created, the controller will create the corresponding `ReplicationSource` CR in the protected namespace. VolSync watches for the creation of `ReplicationSource` CR and copies the PVC data to the restic repository mentioned in the `restic-secret`.
```
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: database-source
  namespace: openshift-adp
spec:
  sourcePVC: <pvc_name>
  trigger:
    manual: <trigger_name>
  restic:
    pruneIntervalDays: 15
    repository: restic-config
    retain:
      hourly: 1
      daily: 1
      weekly: 1
      monthly: 1
      yearly: 1
    copyMethod: None
```

Similarly, when a DataMoverRestore CR gets created, controller will create a `ReplicationDestination` CR in the protected namespace. VolSync controller copies the PVC data from the restic repository to the protected namespace, which then gets transferred to the user namespace by the controller.

```
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: <protected_namespace>
spec:
  trigger:
    manual: <trigger_name>
  restic:
    destinationPVC: <pvc_name>
    repository: restic-config
    copyMethod: None
```

A status controller is created to watch VolSync CRs. It watches the `ReplicationSource` and`ReplicationDestination` objects and updates VolumeSnapShot CR events.

*Note: Potential feature addition to Velero: A status watch controller for DataMover CRs. This can be used to update Velero Backup/Restore events with the DataMover CR results*

Data mover controller will clean up all controller-created resources after the process is complete.

## Open Questions/ Discussion topics

- We would like to open the forum for discussion on what can be learnt from this working PoC and what can be included for Velero datamover.
- Another approach that we can explore is by extending the CSI plugin to move the CSI snapshots.
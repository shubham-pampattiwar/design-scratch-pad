# Ensure support for backing up resources based on multiple labels

[![hackmd-github-sync-badge](https://hackmd.io/GYmS90yOQ42GcoLaW2S4QQ/badge)](https://hackmd.io/GYmS90yOQ42GcoLaW2S4QQ)




## Abstract
As of today Velero supports filtering of resources based on single label per backup. It is desired that Velero support backing up of resources based on multiple labels (OR logic).

## Background
Currently, Velero's Backup API has a spec field `LabelSelector` which helps in filtering of resources based on a **single** label value per backup request. For instance, if the user specifies the `Backup.Spec.LabelSelector` as `data-protection-app: true`, Velero will grab all the resources that posses this label and perform the backup operation on them. The `LabelSelector` field does not accept more than one labels, and thus if the user want to take backup for resources consisting of a label from a set of labels (label1 OR label2 OR label3) then the user needs to create multiple backups per label rule. It would be really useful if Velero Backup API could respect a set of labels (OR Rule) for a single backup request.

Related Issue: https://github.com/vmware-tanzu/velero/issues/1508

## Goals
- Enable support for backing up resources based on multiple labels (OR Logic) in a single backup config.

## Non Goals
- Enable support for backing up resources based on multiple labels (AND Logic) in a single backup config.

## Use Case/Scenario
Let's say as a Velero user you want to take a backup of secrets, but all these secrets do not have one single consistent label on them. We want to take backup of secrets having any one value in `app=gdpr`, `app=wpa` and `app=ccpa`. Here we would have to create 3 instances of backup for each label rule. This can become cumbersome at scale. 

## High-Level Design
### Addition of `LabelSelectors`(_plural_) spec to Velero Backup API
For Velero to backup resources if they consist of any one label from a set of labels, we would like to add a new spec field `LabelSelectors` which would enable user to specify them. The Velero backup command would somewhat look like:

`Velero create backup multi-label-example --selectors 'app=gdpr', 'app=wpa', 'app=ccpa'`

In the above command (`velero --help` will display),
`--selectors is LabelSelectors  Only backup resources matching any one label from the set of label selectors`

Note: This approach will **not** be changing any current behavior related to Backup API spec `LabelSelector`. Rather we propose that the label in `LabelSelector` spec and labels in `LabelSelectors` should be concatenated  and then used as a criteria(resources matching any one label) to grab the resources to be backedup. 
## Detailed Design
With the Introduction of `LabelSelectors` the BackupSpec will look like:

BackupSpec:
```
type BackupSpec struct {
[...]
// LabelSelectosr is a set of []metav1.LabelSelector to filter with
// when adding individual objects to the backup. Resources matching any one
// label from the set of labels will be added to the backup. If empty
// or nil, all objects are included. Optional.
// +optional
LabelSelectors []\*metav1.LabelSelector
[...]
}
```

The logic to collect resources to be backup up for a particular backup will be updated in the `backup/item_collector.go` around [here](https://github.com/vmware-tanzu/velero/blob/574baeb3c920f97b47985ec3957debdc70bcd5f8/pkg/backup/item_collector.go#L294).
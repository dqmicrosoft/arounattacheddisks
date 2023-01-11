# ARO ORPHAN DISKS

### How to delete Unattached Disks in ARO

The disks can be unattached and cannot be deleted if you have created them with the reclaim policy set to Retain.

Even though you have admin permissions you cannot delete the disks because they are managed by the ARO cluster.

To delete them you need to create a PV with the disk unttached and bound it to a PVC. The PV must be using the Storage Class with the reclaim policy set to delete.

## Reproducing Error

I have created one Storage Class to reproduce the error and another to delete the disk (managed-retain and managed-delete).

```sh
> oc get sc
NAME                        PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
managed-csi                 disk.csi.azure.com         Delete          WaitForFirstConsumer   true                   21h
managed-delete              kubernetes.io/azure-disk   Delete          Immediate              true                   2m54s
managed-premium (default)   kubernetes.io/azure-disk   Delete          WaitForFirstConsumer   true                   21h
managed-retain              kubernetes.io/azure-disk   Retain          Immediate              true                   2m43s
```


Let's create the disk with the retain policy set to Retain. And we can see the disk was created:oc delete
```sh
oc apply -f pvc-retain.yaml
```
We get tthe pvc bounded with a pv
```sh
> oc get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
pvc-retain   Bound    pvc-d0f8ae0f-f5bb-4ca4-b4f3-47849480739a   10Gi       RWO            managed-retain   13s

> az disk list --resource-group aro-t4madwlq | grep -v master | grep -v worker
Name                                                               ResourceGroup    Location    Zones    Sku          SizeGb    ProvisioningState    OsType
-----------------------------------------------------------------  ---------------  ----------  -------  -----------  --------  -------------------  --------
arocluster-jkpm9-dynamic-pvc-d0f8ae0f-f5bb-4ca4-b4f3-47849480739a  aro-t4madwlq     westeurope  1        Premium_LRS  10        Succeeded
```

After we delete the PVC we can see the disk still exists and also that it has no owner

```sh
> oc delete pvc pvc-retain
persistentvolumeclaim "pvc-retain" deleted

> oc delete pv pvc-d0f8ae0f-f5bb-4ca4-b4f3-47849480739a
persistentvolume "pvc-d0f8ae0f-f5bb-4ca4-b4f3-47849480739a" deleted

> az disk list --resource-group aro-t4madwlq | grep -v master | grep -v worker
Name                                                               ResourceGroup    Location    Zones    Sku          SizeGb    ProvisioningState    OsType
-----------------------------------------------------------------  ---------------  ----------  -------  -----------  --------  -------------------  --------
arocluster-jkpm9-dynamic-pvc-d0f8ae0f-f5bb-4ca4-b4f3-47849480739a  aro-t4madwlq     westeurope  1        Premium_LRS  10        Succeeded

```

![[Pasted image 20230111155425.png]]

### The Solution

We create a new Storage Class with the Reclaim Policy set to Delete

```sh
> oc apply -f storage-class-delete.yaml
storageclass.storage.k8s.io/managed-delete created
```

We create a new PV but this time we will pass the ID of the disk 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/bound-by-controller: "yes"
    pv.kubernetes.io/provisioned-by: kubernetes.io/azure-disk
    volumehelper.VolumeDynamicallyCreatedByKey: azure-disk-dynamic-provisioner
  name: pv-to-delete
spec:
  accessModes:
  - ReadWriteOnce
  azureDisk:
    cachingMode: ReadOnly
    diskName: arocluster-jkpm9-dynamic-pvc-d0f8ae0f-f5bb-4ca4-b4f3-47849480739a
    diskURI: /subscriptions/89c0f62c-2a04-445a-9830-6d3e02685678/resourceGroups/aro-t4madwlq/providers/Microsoft.Compute/disks/arocluster-jkpm9-dynamic-pvc-d0f8ae0f-f5bb-4ca4-b4f3-47849480739a
    fsType: ""
    kind: Managed
    readOnly: false
  capacity:
    storage: 10Gi
  persistentVolumeReclaimPolicy: Delete
  storageClassName: managed-delete
  volumeMode: Filesystem
```

**_NOTE: Change the variables according to your disk setup_**

We also configure the PVC to be bound to the PV we will create

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-to-delete
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: managed-delete
  volumeName: pv-to-delete
```

Ands we apply both

```sh
> oc apply -f pv-to-delete-disk.yaml
> > oc apply -f pvc-to-delete-disk.yaml
```

We get this result
```sh
> oc get pvc
NAME            STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS     AGE
pvc-to-delete   Bound    pv-to-delete   10Gi       RWO            managed-delete   70s
```

We delete the PVC and we can see the disk will be deleted too

```sh
> oc delete pvc pvc-to-delete
persistentvolumeclaim "pvc-to-delete" deleted
```

```sh
> az disk list --resource-group aro-t4madwlq | grep -v master | grep -v worker
Name                                              ResourceGroup    Location    Zones    Sku          OsType    SizeGb    ProvisioningState
------------------------------------------------  ---------------  ----------  -------  -----------  --------  --------  -------------------
```

## Graceful uninstall of Kubera

By deleting cluster from Director Online will delete all pods under `maya-system` namespace. But it doesnot remove components are `openebs` namespace and CRDs related to OpenEBS and Director Online. This can be done by running the following commands.

Delete all SCs

```
kubectl get sc | grep -v "standard" | awk '{print $1}' | grep -v "NAME" |  xargs kubectl delete sc
```

Delete OpenEBS namespace

```
kubectl delete ns openebs
```

Delete all CRDs naming with OpenEBS, Volumesnapshot, Velero and MayaData.

```
kubectl get crds | grep -i "openebs\|volumesnapshot\|velero\|mayadata\|csi" | awk '{print $1}' | xargs kubectl delete crds
```

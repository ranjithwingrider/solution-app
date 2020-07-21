This will help to handle a scale down/ Node down scenario when a Local PV volume has been used as the persistent storage volume for your stateful application, in this case Prometheus.

## Before scale down

### Get the details of Pods and PVCs are running under `monitoring` namespace

```
$ kubectl get pod -n monitoring -o wide

NAME                                             READY   STATUS    RESTARTS   AGE   IP            NODE                                        NOMINATED NODE   READINESS GATES
alertmanager-new-alertmanager-0                  2/2     Running   0          70m   10.20.0.16    gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
alertmanager-new-alertmanager-1                  2/2     Running   0          70m   10.20.2.21    gke-prometheus-default-pool-8ba1a274-zhk3   <none>           <none>
alertmanager-new-alertmanager-2                  2/2     Running   0          70m   10.20.1.12    gke-prometheus-default-pool-8ba1a274-h2bt   <none>           <none>
new-operator-5bfb4dc869-tkzxl                    1/1     Running   0          71m   10.20.1.11    gke-prometheus-default-pool-8ba1a274-h2bt   <none>           <none>
prometheus-grafana-85b4dbb556-lmsdr              2/2     Running   0          71m   10.20.0.14    gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
prometheus-kube-state-metrics-6df5d44568-b22bh   1/1     Running   0          71m   10.20.0.15    gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
prometheus-new-prometheus-0                      3/3     Running   1          70m   10.20.1.13    gke-prometheus-default-pool-8ba1a274-h2bt   <none>           <none>
prometheus-new-prometheus-1                      3/3     Running   1          70m   10.20.2.22    gke-prometheus-default-pool-8ba1a274-zhk3   <none>           <none>
prometheus-new-prometheus-2                      3/3     Running   1          70m   10.20.0.17    gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
prometheus-prometheus-node-exporter-dklvk        1/1     Running   0          71m   10.128.0.61   gke-prometheus-default-pool-8ba1a274-h2bt   <none>           <none>
prometheus-prometheus-node-exporter-pd2nt        1/1     Running   0          71m   10.128.0.62   gke-prometheus-default-pool-8ba1a274-zhk3   <none>           <none>
prometheus-prometheus-node-exporter-rmprg        1/1     Running   0          71m   10.128.0.59   gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
```

```
$ kubectl get pvc -n monitoring

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-0   Bound    pvc-b947139a-6e60-4d76-bb6b-95f1803f4882   100Gi      RWO            openebs-device   71m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-1   Bound    pvc-76426ffd-fec5-41a2-aa19-20a91349b0c9   100Gi      RWO            openebs-device   71m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-2   Bound    pvc-b55023a0-7f94-41a1-8f37-108b2f432297   100Gi      RWO            openebs-device   71m
prometheus-new-prometheus-db-prometheus-new-prometheus-0           Bound    pvc-4829833c-994e-445c-9b16-4a40d81f95b1   100Gi      RWO            openebs-device   70m
prometheus-new-prometheus-db-prometheus-new-prometheus-1           Bound    pvc-4580c6df-759d-4a0a-9459-b9737a01f10b   100Gi      RWO            openebs-device   70m
prometheus-new-prometheus-db-prometheus-new-prometheus-2           Bound    pvc-df1e8ec1-7de0-427e-9bdb-03014265e608   100Gi      RWO            openebs-device   70m
```

### Get the details of Nodes
```
$ kubectl get node -o wide

NAME                                        STATUS   ROLES    AGE    VERSION          INTERNAL-IP   EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
gke-prometheus-default-pool-8ba1a274-k6cj   Ready    <none>   167m   v1.16.11-gke.5   10.128.0.59   35.225.65.78     Ubuntu 18.04.4 LTS   5.3.0-1016-gke   docker://19.3.2
gke-prometheus-default-pool-8ba1a274-h2bt   Ready    <none>   167m   v1.16.11-gke.5   10.128.0.61   34.66.41.150     Ubuntu 18.04.4 LTS   5.3.0-1016-gke   docker://19.3.2
gke-prometheus-default-pool-8ba1a274-zhk3   Ready    <none>   167m   v1.16.11-gke.5   10.128.0.62   35.193.112.228   Ubuntu 18.04.4 LTS   5.3.0-1016-gke   docker://19.3.2
```

### Get the details of nodes with labels
```
$ kubectl get node --show-labels

NAME                                        STATUS   ROLES    AGE    VERSION          LABELS
gke-prometheus-default-pool-8ba1a274-k6cj   Ready    <none>   169m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-k6cj,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,node=prometheus,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-k6cj
gke-prometheus-default-pool-8ba1a274-h2bt   Ready    <none>   169m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-h2bt,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,node=prometheus,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-h2bt
gke-prometheus-default-pool-8ba1a274-zhk3   Ready    <none>   169m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-zhk3,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,node=prometheus,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-zhk3
```

### Get the details Blockdevices attached to each node
```
$ kubectl get bd -n openebs

NAME                                           NODENAME                                    SIZE           CLAIMSTATE   STATUS   AGE
blockdevice-4f51859193d333687a873af7acf8ad78   gke-prometheus-default-pool-8ba1a274-h2bt   107374182400   Claimed      Active   94m
blockdevice-630ae186f095cd94d9158bdaa0005ae4   gke-prometheus-default-pool-8ba1a274-k6cj   107374182400   Claimed      Active   94m
blockdevice-747a07ffae7a6a53762b3ce262c3307a   gke-prometheus-default-pool-8ba1a274-zhk3   107374182400   Claimed      Active   94m
blockdevice-967d7816c2a2d73b91c8c6310dc70465   gke-prometheus-default-pool-8ba1a274-k6cj   107374182400   Claimed      Active   94m
blockdevice-ddfc782ea661fc9007a896438f483e3d   gke-prometheus-default-pool-8ba1a274-zhk3   107374182400   Claimed      Active   94m
blockdevice-e5265da8a790a2374758ec4600cd4bd7   gke-prometheus-default-pool-8ba1a274-h2bt   107374182400   Claimed      Active   94m
```



## After scale down

### Get the details of nodes with labels

```
$ kubectl get node --show-labels

NAME                                        STATUS   ROLES    AGE    VERSION          LABELS
gke-prometheus-default-pool-8ba1a274-k6cj   Ready    <none>   173m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-k6cj,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,node=prometheus,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-k6cj
gke-prometheus-default-pool-8ba1a274-zhk3   Ready    <none>   173m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-zhk3,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,node=prometheus,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-zhk3
```

### Get the details of Blockdevices 
```
$ kubectl get bd -n openebs

NAME                                           NODENAME                                    SIZE           CLAIMSTATE   STATUS    AGE
blockdevice-4f51859193d333687a873af7acf8ad78   gke-prometheus-default-pool-8ba1a274-h2bt   107374182400   Claimed      Unknown   101m
blockdevice-630ae186f095cd94d9158bdaa0005ae4   gke-prometheus-default-pool-8ba1a274-k6cj   107374182400   Claimed      Active    100m
blockdevice-747a07ffae7a6a53762b3ce262c3307a   gke-prometheus-default-pool-8ba1a274-zhk3   107374182400   Claimed      Active    100m
blockdevice-967d7816c2a2d73b91c8c6310dc70465   gke-prometheus-default-pool-8ba1a274-k6cj   107374182400   Claimed      Active    101m
blockdevice-ddfc782ea661fc9007a896438f483e3d   gke-prometheus-default-pool-8ba1a274-zhk3   107374182400   Claimed      Active    101m
blockdevice-e5265da8a790a2374758ec4600cd4bd7   gke-prometheus-default-pool-8ba1a274-h2bt   107374182400   Claimed      Active    100m
```

### Get the details of PVCs

```
$ kubectl get pvc -n monitoring

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-0   Bound    pvc-b947139a-6e60-4d76-bb6b-95f1803f4882   100Gi      RWO            openebs-device   78m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-1   Bound    pvc-76426ffd-fec5-41a2-aa19-20a91349b0c9   100Gi      RWO            openebs-device   78m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-2   Bound    pvc-b55023a0-7f94-41a1-8f37-108b2f432297   100Gi      RWO            openebs-device   78m
prometheus-new-prometheus-db-prometheus-new-prometheus-0           Bound    pvc-4829833c-994e-445c-9b16-4a40d81f95b1   100Gi      RWO            openebs-device   77m
prometheus-new-prometheus-db-prometheus-new-prometheus-1           Bound    pvc-4580c6df-759d-4a0a-9459-b9737a01f10b   100Gi      RWO            openebs-device   77m
prometheus-new-prometheus-db-prometheus-new-prometheus-2           Bound    pvc-df1e8ec1-7de0-427e-9bdb-03014265e608   100Gi      RWO            openebs-device   77m
```

### Get the details of Pods

```
$ kubectl get pod -n monitoring -o wide

NAME                                             READY   STATUS    RESTARTS   AGE     IP            NODE                                        NOMINATED NODE   READINESS GATES
alertmanager-new-alertmanager-0                  2/2     Running   0          78m     10.20.0.16    gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
alertmanager-new-alertmanager-1                  2/2     Running   0          78m     10.20.2.21    gke-prometheus-default-pool-8ba1a274-zhk3   <none>           <none>
alertmanager-new-alertmanager-2                  0/2     Pending   0          4m29s   <none>        <none>                                      <none>           <none>
new-operator-5bfb4dc869-vcrzs                    1/1     Running   0          4m36s   10.20.2.26    gke-prometheus-default-pool-8ba1a274-zhk3   <none>           <none>
prometheus-grafana-85b4dbb556-lmsdr              2/2     Running   0          78m     10.20.0.14    gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
prometheus-kube-state-metrics-6df5d44568-b22bh   1/1     Running   0          78m     10.20.0.15    gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
prometheus-new-prometheus-0                      0/3     Pending   0          4m29s   <none>        <none>                                      <none>           <none>
prometheus-new-prometheus-1                      3/3     Running   1          78m     10.20.2.22    gke-prometheus-default-pool-8ba1a274-zhk3   <none>           <none>
prometheus-new-prometheus-2                      3/3     Running   1          78m     10.20.0.17    gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
prometheus-prometheus-node-exporter-pd2nt        1/1     Running   0          78m     10.128.0.62   gke-prometheus-default-pool-8ba1a274-zhk3   <none>           <none>
prometheus-prometheus-node-exporter-rmprg        1/1     Running   0          78m     10.128.0.59   gke-prometheus-default-pool-8ba1a274-k6cj   <none>           <none>
```


## After scale up 

### Labellng new node

```
$ kubectl label node gke-prometheus-default-pool-8ba1a274-h2bt mayadata.io/control-plane=true
node/gke-prometheus-default-pool-8ba1a274-h2bt labeled

$ kubectl label node gke-prometheus-default-pool-8ba1a274-h2bt mayadata.io/data-plane=true
node/gke-prometheus-default-pool-8ba1a274-h2bt labeled

$ kubectl label node gke-prometheus-default-pool-8ba1a274-h2bt node=prometheus
node/gke-prometheus-default-pool-8ba1a274-h2bt labeled
```

### Check labels of Nodes

```
$ kubectl get node --show-labels

NAME                                        STATUS   ROLES    AGE     VERSION          LABELS
gke-prometheus-default-pool-8ba1a274-h2bt   Ready    <none>   6m17s   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-h2bt,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/date-plane=true,node=prometheus,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-h2bt
gke-prometheus-default-pool-8ba1a274-k6cj   Ready    <none>   3h14m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-k6cj,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,node=prometheus,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-k6cj
gke-prometheus-default-pool-8ba1a274-zhk3   Ready    <none>   3h14m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-zhk3,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,node=prometheus,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-zhk3
```

### Ensure the NDM pod is running on new node

```
$ kubectl get pod -n openebs
NAME                                              READY   STATUS    RESTARTS   AGE
cspc-operator-64b59655db-8hmb6                    1/1     Running   0          167m
cvc-operator-5cb75f7cd6-wlmn8                     1/1     Running   0          167m
maya-apiserver-77975bbdcf-6nrjr                   1/1     Running   2          167m
openebs-admission-server-66b6b8d44f-p9rgk         1/1     Running   0          167m
openebs-cstor-admission-server-68d6bd8f6d-xjcmc   1/1     Running   0          167m
openebs-localpv-provisioner-6cd4859c7c-gj5b7      1/1     Running   0          23m
openebs-ndm-59x7f                                 1/1     Running   0          14s
openebs-ndm-gtppd                                 1/1     Running   0          167m
openebs-ndm-operator-f58cc87cf-nrpn6              1/1     Running   1          167m
openebs-ndm-p96mx                                 1/1     Running   0          167m
openebs-provisioner-7b7d644c47-zqjhs              1/1     Running   0          167m
openebs-snapshot-operator-76d499b696-jj8pj        2/2     Running   0          167m
```

### Attach old disks to new node

```
gcloud compute instances attach-disk gke-prometheus-default-pool-8ba1a274-h2bt --disk prometheus-disk2 --device-name prometheus-disk2 --zone=us-central1-c
gcloud compute instances attach-disk gke-prometheus-default-pool-8ba1a274-h2bt --disk prometheus-disk5 --device-name prometheus-disk5 --zone=us-central1-c
```

### Verify if new BDs are atatched to new node

```
$ kubectl get bd -n openebs
NAME                                           NODENAME                                    SIZE           CLAIMSTATE   STATUS   AGE
blockdevice-4f51859193d333687a873af7acf8ad78   gke-prometheus-default-pool-8ba1a274-h2bt   107374182400   Claimed      Active   124m
blockdevice-630ae186f095cd94d9158bdaa0005ae4   gke-prometheus-default-pool-8ba1a274-k6cj   107374182400   Claimed      Active   123m
blockdevice-747a07ffae7a6a53762b3ce262c3307a   gke-prometheus-default-pool-8ba1a274-zhk3   107374182400   Claimed      Active   123m
blockdevice-967d7816c2a2d73b91c8c6310dc70465   gke-prometheus-default-pool-8ba1a274-k6cj   107374182400   Claimed      Active   124m
blockdevice-ddfc782ea661fc9007a896438f483e3d   gke-prometheus-default-pool-8ba1a274-zhk3   107374182400   Claimed      Active   123m
blockdevice-e5265da8a790a2374758ec4600cd4bd7   gke-prometheus-default-pool-8ba1a274-h2bt   107374182400   Claimed      Active   123m
```

### Get the details of Pods and PVCs are running under `monitoring` namespace

```
$ kubectl get pod -n monitoring

NAME                                             READY   STATUS    RESTARTS   AGE
alertmanager-new-alertmanager-0                  2/2     Running   0          101m
alertmanager-new-alertmanager-1                  2/2     Running   0          101m
alertmanager-new-alertmanager-2                  0/2     Pending   0          27m
new-operator-5bfb4dc869-vcrzs                    1/1     Running   0          27m
prometheus-grafana-85b4dbb556-lmsdr              2/2     Running   0          101m
prometheus-kube-state-metrics-6df5d44568-b22bh   1/1     Running   0          101m
prometheus-new-prometheus-0                      0/3     Pending   0          27m
prometheus-new-prometheus-1                      3/3     Running   1          101m
prometheus-new-prometheus-2                      3/3     Running   1          101m
prometheus-prometheus-node-exporter-pd2nt        1/1     Running   0          101m
prometheus-prometheus-node-exporter-rmprg        1/1     Running   0          101m
prometheus-prometheus-node-exporter-x2tms        1/1     Running   0          9m15s
```

```
$ kubectl get pvc -n monitoring

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-0   Bound    pvc-b947139a-6e60-4d76-bb6b-95f1803f4882   100Gi      RWO            openebs-device   101m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-1   Bound    pvc-76426ffd-fec5-41a2-aa19-20a91349b0c9   100Gi      RWO            openebs-device   101m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-2   Bound    pvc-b55023a0-7f94-41a1-8f37-108b2f432297   100Gi      RWO            openebs-device   101m
prometheus-new-prometheus-db-prometheus-new-prometheus-0           Bound    pvc-4829833c-994e-445c-9b16-4a40d81f95b1   100Gi      RWO            openebs-device   101m
prometheus-new-prometheus-db-prometheus-new-prometheus-1           Bound    pvc-4580c6df-759d-4a0a-9459-b9737a01f10b   100Gi      RWO            openebs-device   101m
prometheus-new-prometheus-db-prometheus-new-prometheus-2           Bound    pvc-df1e8ec1-7de0-427e-9bdb-03014265e608   100Gi      RWO            openebs-device   101m
```

### Delete the PVCs associated to the pending pods one by one

```
$ kubectl delete pvc -n monitoring alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-2 prometheus-new-prometheus-db-prometheus-new-prometheus-0

persistentvolumeclaim "alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-2" deleted
persistentvolumeclaim "prometheus-new-prometheus-db-prometheus-new-prometheus-0" deleted
```

### Delete the old PVCs associated to `Pending` state

```
$ kubectl get pvc -n monitoring

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-0   Bound    pvc-b947139a-6e60-4d76-bb6b-95f1803f4882   100Gi      RWO            openebs-device   102m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-1   Bound    pvc-76426ffd-fec5-41a2-aa19-20a91349b0c9   100Gi      RWO            openebs-device   102m
prometheus-new-prometheus-db-prometheus-new-prometheus-1           Bound    pvc-4580c6df-759d-4a0a-9459-b9737a01f10b   100Gi      RWO            openebs-device   102m
prometheus-new-prometheus-db-prometheus-new-prometheus-2           Bound    pvc-df1e8ec1-7de0-427e-9bdb-03014265e608   100Gi      RWO            openebs-device   102m
```

### Delete the old Pods which is in `Pending` state 

```
$ kubectl delete pod -n monitoring alertmanager-new-alertmanager-2
pod "alertmanager-new-alertmanager-2" deleted

$ kubectl delete pod -n monitoring prometheus-new-prometheus-0
pod "prometheus-new-prometheus-0" deleted
```

### Verify new Pods,PVCs are running under `monitoring` namespace

```
$ kubectl get pvc -n monitoring

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-0   Bound    pvc-b947139a-6e60-4d76-bb6b-95f1803f4882   100Gi      RWO            openebs-device   103m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-1   Bound    pvc-76426ffd-fec5-41a2-aa19-20a91349b0c9   100Gi      RWO            openebs-device   103m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-2   Bound    pvc-c59600c9-9904-4d6e-b30c-f1fd95c1af16   100Gi      RWO            openebs-device   38s
prometheus-new-prometheus-db-prometheus-new-prometheus-0           Bound    pvc-801736a3-2d59-486b-945e-a461da5ee4a7   100Gi      RWO            openebs-device   30s
prometheus-new-prometheus-db-prometheus-new-prometheus-1           Bound    pvc-4580c6df-759d-4a0a-9459-b9737a01f10b   100Gi      RWO            openebs-device   103m
prometheus-new-prometheus-db-prometheus-new-prometheus-2           Bound    pvc-df1e8ec1-7de0-427e-9bdb-03014265e608   100Gi      RWO            openebs-device   103m
```

```
$ kubectl get pod -n monitoring

NAME                                             READY   STATUS    RESTARTS   AGE
alertmanager-new-alertmanager-0                  2/2     Running   0          103m
alertmanager-new-alertmanager-1                  2/2     Running   0          103m
alertmanager-new-alertmanager-2                  2/2     Running   0          45s
new-operator-5bfb4dc869-vcrzs                    1/1     Running   0          30m
prometheus-grafana-85b4dbb556-lmsdr              2/2     Running   0          103m
prometheus-kube-state-metrics-6df5d44568-b22bh   1/1     Running   0          103m
prometheus-new-prometheus-0                      3/3     Running   1          37s
prometheus-new-prometheus-1                      3/3     Running   1          103m
prometheus-new-prometheus-2                      3/3     Running   1          103m
prometheus-prometheus-node-exporter-pd2nt        1/1     Running   0          103m
prometheus-prometheus-node-exporter-rmprg        1/1     Running   0          103m
prometheus-prometheus-node-exporter-x2tms        1/1     Running   0          11m
```

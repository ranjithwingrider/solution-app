This will help to handle a scale down/ Node down scenario when a Local PV volume has been used as the persistent storage volume for your stateful application, in this case Prometheus.

### Get the details of node before the scale dowm

```
kubectl get node --show-labels
```
Sample output

```
NAME                                        STATUS   ROLES    AGE   VERSION          LABELS
gke-prometheus-default-pool-8ba1a274-k6cj   Ready    <none>   80m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-k6cj,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-k6cj
gke-prometheus-default-pool-8ba1a274-t76h   Ready    <none>   80m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-t76h,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-t76h
gke-prometheus-default-pool-8ba1a274-zhk3   Ready    <none>   80m   v1.16.11-gke.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=ubuntu,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-c,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-prometheus-default-pool-8ba1a274-zhk3,kubernetes.io/os=linux,mayadata.io/control-plane=true,mayadata.io/data-plane=true,topology.cstor.openebs.io/nodeName=gke-prometheus-default-pool-8ba1a274-zhk3
```

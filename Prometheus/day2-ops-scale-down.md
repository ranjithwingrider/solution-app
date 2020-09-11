This will help to handle a scale down/ Node down scenario when a Local PV device has been used as the persistent storage volume for your stateful application, in this case Kudo based Cassandra.

## Before scale down

### Get the details of Pods, PVCs, SVC and PVs are running under `cassandra` namespace

```
$ kubectl get pod -n cassandra

NAME                       READY   STATUS    RESTARTS   AGE    IP               NODE                                            NOMINATED NODE   READINESS GATES
cassandra-openebs-node-0   1/1     Running   0          123m   192.168.21.10    ip-192-168-22-232.ap-south-1.compute.internal   <none>           <none>
cassandra-openebs-node-1   1/1     Running   0          122m   192.168.33.255   ip-192-168-60-228.ap-south-1.compute.internal   <none>           <none>
cassandra-openebs-node-2   1/1     Running   0          121m   192.168.74.145   ip-192-168-82-140.ap-south-1.compute.internal   <none>           <none>
```

```
$ kubectl get pvc,svc -n cassandra

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-0   Bound    pvc-22b0a740-f58d-41d6-8520-325fc7266a4f   20Gi       RWO            openebs-device   119m
persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-1   Bound    pvc-0d36674e-9094-4dca-889c-40d62bc6bb10   20Gi       RWO            openebs-device   118m
persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-2   Bound    pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827   20Gi       RWO            openebs-device   117m

NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
service/cassandra-openebs-svc   ClusterIP   10.100.9.95   <none>        7000/TCP,7001/TCP,9042/TCP   119m

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                  STORAGECLASS     REASON   AGE
pvc-0d36674e-9094-4dca-889c-40d62bc6bb10   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-1   openebs-device            118m
pvc-22b0a740-f58d-41d6-8520-325fc7266a4f   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-0   openebs-device            119m
pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-2   openebs-device            117m
```

### Get the details of Nodes
```
$ kubectl get node --show-labels

NAME                                            STATUS   ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-192-168-22-232.ap-south-1.compute.internal   Ready    <none>   3h28m   v1.16.9   192.168.22.232   3.7.255.62       Ubuntu 18.04.4 LTS   5.3.0-1030-aws   docker://17.3.2
ip-192-168-60-228.ap-south-1.compute.internal   Ready    <none>   3h28m   v1.16.9   192.168.60.228   15.207.100.219   Ubuntu 18.04.4 LTS   5.3.0-1030-aws   docker://17.3.2
ip-192-168-82-140.ap-south-1.compute.internal   Ready    <none>   3h28m   v1.16.9   192.168.82.140   13.127.209.74    Ubuntu 18.04.4 LTS   5.3.0-1030-aws   docker://17.3.2


```

### Get the details of nodes with labels
```
$ kubectl get node --show-labels

NAME                                            STATUS   ROLES    AGE     VERSION   LABELS
ip-192-168-22-232.ap-south-1.compute.internal   Ready    <none>   3h28m   v1.16.9   alpha.eksctl.io/cluster-name=ranjith-eks3,alpha.eksctl.io/instance-id=i-056a9695e63d2efd4,alpha.eksctl.io/nodegroup-name=standard-workers,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=ap-south-1,failure-domain.beta.kubernetes.io/zone=ap-south-1c,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-22-232,kubernetes.io/os=linux
ip-192-168-60-228.ap-south-1.compute.internal   Ready    <none>   3h28m   v1.16.9   alpha.eksctl.io/cluster-name=ranjith-eks3,alpha.eksctl.io/instance-id=i-046af28d7ea9027e1,alpha.eksctl.io/nodegroup-name=standard-workers,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=ap-south-1,failure-domain.beta.kubernetes.io/zone=ap-south-1b,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-60-228,kubernetes.io/os=linux
ip-192-168-82-140.ap-south-1.compute.internal   Ready    <none>   3h28m   v1.16.9   alpha.eksctl.io/cluster-name=ranjith-eks3,alpha.eksctl.io/instance-id=i-07ae53e185aa718c2,alpha.eksctl.io/nodegroup-name=standard-workers,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=ap-south-1,failure-domain.beta.kubernetes.io/zone=ap-south-1a,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-82-140,kubernetes.io/os=linux
```

### Get the details Blockdevices attached to each node
```
$ kubectl get bd -n openebs

NAME                                           NODENAME                                        SIZE          CLAIMSTATE   STATUS   AGE
blockdevice-82d4e91902529bd6d718f8fbe956274b   ip-192-168-22-232.ap-south-1.compute.internal   42949672960   Claimed      Active   123m
blockdevice-978ffcc0ee3c7b8bc709b5fef0b25953   ip-192-168-60-228.ap-south-1.compute.internal   42949672960   Claimed      Active   123m
blockdevice-ce1a004f2f7ba2fa603a2e8e0331aea0   ip-192-168-82-140.ap-south-1.compute.internal   42949672960   Claimed      Active   123m
```

### Add some data into the the Database

```
kubectl exec -it cassandra-openebs-node-0 bash -n cassandra
cassandra@cassandra-openebs-node-0:/$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address         Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.21.10   196.33 KiB  256          67.4%             29bb331f-67a2-4440-86dc-32540aa40bc1  rack1
UN  192.168.33.255  208.2 KiB  256          66.3%             d16694a9-18d3-4625-a6b7-346ef09545a0  rack1
UN  192.168.74.145  231.38 KiB  256          66.4%             db10a0d7-2b42-4e67-b048-b6dca0ded0a6  rack1

cassandra@cassandra-openebs-node-0:/$ cqlsh --version
cqlsh 5.0.1

cassandra@cassandra-openebs-node-0:/$ cqlsh cassandra-openebs-svc.cassandra.svc.cluster.local
Connected to cassandra-openebs at cassandra-openebs-svc.cassandra.svc.cluster.local:9042.
[cqlsh 5.0.1 | Cassandra 3.11.6 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> create keyspace dev
   ... with replication = {'class':'SimpleStrategy','replication_factor':1};

cqlsh> use dev;

cqlsh:dev> create table emp (empid int primary key,
       ... emp_first varchar, emp_last varchar, emp_dept varchar);

cqlsh:dev> insert into emp (empid, emp_first, emp_last, emp_dept)
       ... values (1,'fred','smith','eng');

cqlsh:dev> select * from emp;

 empid | emp_dept | emp_first | emp_last
-------+----------+-----------+----------
     1 |      eng |      fred |    smith

(1 rows)

cqlsh:dev> update emp set emp_dept = 'fin' where empid = 1;

cqlsh:dev> select * from emp;

 empid | emp_dept | emp_first | emp_last
-------+----------+-----------+----------
     1 |      fin |      fred |    smith

(1 rows)

cqlsh:dev> insert into emp (empid, emp_first, emp_last, emp_dept)
       ... values (2,'eon','morgan','sales');

cqlsh:dev> insert into emp (empid, emp_first, emp_last, emp_dept) values (3,'Sree','Ram','qa');

cqlsh:dev> select * from emp;

 empid | emp_dept | emp_first | emp_last
-------+----------+-----------+----------
     1 |      fin |      fred |    smith
     2 |    sales |       eon |   morgan
     3 |       qa |     Ashok |    Kumar

(3 rows)
cqlsh:dev> exit
```

## After scale down

### Get the details of nodes

```
$ kubectl get node
No resources found in default namespace.
```

### Get the details of Pods,PVCs, SVC and PVs

```
root@ranjith-m:~# kubectl get pod,pvc,svc -n cassandra
NAME                           READY   STATUS    RESTARTS   AGE
pod/cassandra-openebs-node-0   0/1     Pending   0          99s

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-0   Bound    pvc-22b0a740-f58d-41d6-8520-325fc7266a4f   20Gi       RWO            openebs-device   131m
persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-1   Bound    pvc-0d36674e-9094-4dca-889c-40d62bc6bb10   20Gi       RWO            openebs-device   130m
persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-2   Bound    pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827   20Gi       RWO            openebs-device   129m

NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
service/cassandra-openebs-svc   ClusterIP   10.100.9.95   <none>        7000/TCP,7001/TCP,9042/TCP   131m

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                  STORAGECLASS     REASON   AGE
pvc-0d36674e-9094-4dca-889c-40d62bc6bb10   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-1   openebs-device            130m
pvc-22b0a740-f58d-41d6-8520-325fc7266a4f   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-0   openebs-device            131m
pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-2   openebs-device            129m
```

## After scale up 

1. Label the new nodes with the same custom label used in the `nodeSelector` field in the STS app. This is optional and only applicable if any such fields are added in Application.

2. Attach the disks to any node in the same zone. There is no specific order for the disk attachment. Note down the device name and node name where the particular device is getting attached. This information is required for Step5.
   In my case, I have identified the list of Instances and Volume details in the specified region. I have already give my default region using `aws configure`.
   ```
   $ aws ec2 describe-instances --query 'Reservations[*].Instances[*].{Instances:InstanceId,AZ:Placement.AvailabilityZone}' --output table
   
   |           DescribeInstances          |
   +--------------+-----------------------+
   |      AZ      |       Instances       |
   +--------------+-----------------------+
   |  ap-south-1c |  i-06001f0f449dd1e45  |
   |  ap-south-1a |  i-08a08af7384adfcd5  |
   |  ap-south-1b |  i-0d10d0e48dec8c173  |
   +--------------+-----------------------+

   $ aws ec2 describe-volumes --filters Name=status,Values=available --query "Volumes[*].{VolId:VolumeId,AZ:AvailabilityZone,Size:Size}" --output table
   
   --------------------------------------------------
   |                 DescribeVolumes                |
   +--------------+-------+-------------------------+
   |      AZ      | Size  |          VolId          |
   +--------------+-------+-------------------------+
   |  ap-south-1b |  40   |  vol-01085b2eb9395d2bc  |
   |  ap-south-1c |  40   |  vol-086bbc80a9878906e  |
   |  ap-south-1a |  40   |  vol-0a959c7d5ca02bfa5  |
   +--------------+-------+-------------------------+

   $ aws ec2 attach-volume --volume-id vol-0a959c7d5ca02bfa5 --instance-id i-08a08af7384adfcd5 --device /dev/sdf
  
   $ aws ec2 attach-volume --volume-id vol-01085b2eb9395d2bc --instance-id i-0d10d0e48dec8c173 --device /dev/sdf
  
   $ aws ec2 attach-volume --volume-id vol-086bbc80a9878906e --instance-id i-06001f0f449dd1e45 --device /dev/sdf
   ```
     
3. This step is only apllicable for the following cases:
 
   - If OpenEBS is installed using MayaData Kubera. OpenEBS pods will be in running state only if you label your new nodes with the following labels.  If you installed OpenEBS using `helm` or `kubectl`, then skip this step.
     ```   
     $ kubectl label node <Node_name> mayadata.io/control-plane=true
     
     $ kubectl label node <Node_name> mayadata.io/data-plane=true
     ```
   - If your application is used some custom label as `nodeSelector`, the you should label the nodes accordingly.
     ```
     $ kubectl label node <Node_name> <custom-label-used-in-nodeSelector-field-in-Application>
     ```
   In my case, I have used Kudo Cassandra and I do not required to do any labelling of Nodes.

    ```
    $ kubectl get node --show-labels

    NAME                                           STATUS   ROLES    AGE   VERSION   LABELS
    ip-192-168-24-84.ap-south-1.compute.internal   Ready    <none>   15m   v1.16.9   alpha.eksctl.io/cluster-name=ranjith-eks3,alpha.eksctl.io/instance-id=i-06001f0f449dd1e45,alpha.eksctl.io/nodegroup-name=standard-workers,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=ap-south-1,failure-domain.beta.kubernetes.io/zone=ap-south-1c,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-24-84,kubernetes.io/os=linux
    ip-192-168-47-20.ap-south-1.compute.internal   Ready    <none>   15m   v1.16.9   alpha.eksctl.io/cluster-name=ranjith-eks3,alpha.eksctl.io/instance-id=i-0d10d0e48dec8c173,alpha.eksctl.io/nodegroup-name=standard-workers,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=ap-south-1,failure-domain.beta.kubernetes.io/zone=ap-south-1b,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-47-20,kubernetes.io/os=linux
    ip-192-168-93-98.ap-south-1.compute.internal   Ready    <none>   15m   v1.16.9   alpha.eksctl.io/cluster-name=ranjith-eks3,alpha.eksctl.io/instance-id=i-08a08af7384adfcd5,alpha.eksctl.io/nodegroup-name=standard-workers,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=ap-south-1,failure-domain.beta.kubernetes.io/zone=ap-south-1a,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-93-98,kubernetes.io/os=linux
    ```

4. Ensure the NDM pods and other OpenEBS pods are running on new nodes

   ```
   $ kubectl get pod -n openebs

   ```

5. Verify if new BDs are attched to new nodes

   ```
   $ kubectl get bd -n openebs
   
   NAME                                           NODENAME                                       SIZE          CLAIMSTATE   STATUS   AGE
   blockdevice-82d4e91902529bd6d718f8fbe956274b   ip-192-168-24-84.ap-south-1.compute.internal   42949672960   Claimed      Active   151m
   blockdevice-978ffcc0ee3c7b8bc709b5fef0b25953   ip-192-168-47-20.ap-south-1.compute.internal   42949672960   Claimed      Active   151m
   blockdevice-ce1a004f2f7ba2fa603a2e8e0331aea0   ip-192-168-93-98.ap-south-1.compute.internal   42949672960   Claimed      Active   151m
   ```
6. Get the PV details
   
   ```
   $ kubectl get pv
   
   NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                  STORAGECLASS     REASON    AGE
   pvc-0d36674e-9094-4dca-889c-40d62bc6bb10   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-1   openebs-device              148m
   pvc-22b0a740-f58d-41d6-8520-325fc7266a4f   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-0   openebs-device              149m
   pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-2   openebs-device              147m

   ```
   Copy the YAML spec of all the associated PVs like below
   ```
   $ kubectl get pv pvc-0d36674e-9094-4dca-889c-40d62bc6bb10 -o yaml > pv1.yaml
   $ kubectl get pv pvc-22b0a740-f58d-41d6-8520-325fc7266a4f -o yaml > pv0.yaml
   $ kubectl get pv pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827 -o yaml > pv2.yaml
   ```
7. Modify the above copied YAML specs with new hostname where the disk was attached. The change has to be needed for `value` of `key` `kubernetes.io/hostname`.

8. Remove Finalizers from the identified PVs which are going to be deleted. Once the PVs are modified, delete the PVs using the following commands.
   
   ```
   $ kubectl delete pv pvc-0d36674e-9094-4dca-889c-40d62bc6bb10 pvc-22b0a740-f58d-41d6-8520-325fc7266a4f pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827
   ```
9. Check the status of PVs and application Pods.
   The following shows the PVs are Lost.
   ```
   $ kubectl get pod,pvc -n cassandra
   NAME                           READY   STATUS    RESTARTS   AGE
   pod/cassandra-openebs-node-0   0/1     Pending   0          102m

   NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
   persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-0   Lost     pvc-22b0a740-f58d-41d6-8520-325fc7266a4f   0                         openebs-device   3h51m
   persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-1   Lost     pvc-0d36674e-9094-4dca-889c-40d62bc6bb10   0                         openebs-device   3h50m
   persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-2   Lost     pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827   0                         openebs-device   3h49m
   ```
10. Apply the updated YAML files. 
    
    ```
    $ kubectl apply -f pv0.yaml
    persistentvolume/pvc-22b0a740-f58d-41d6-8520-325fc7266a4f created
    
    $ kubectl apply -f pv1.yaml
    persistentvolume/pvc-0d36674e-9094-4dca-889c-40d62bc6bb10 created
    
    $ kubectl apply -f pv2.yaml
    persistentvolume/pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827 created
    ```
 11. Verify PODs are started `Running` from `Pending` state.
  
     ```
     $ kubectl get pod,pvc,svc -n cassandra
    
     NAME                           READY   STATUS    RESTARTS   AGE
     pod/cassandra-openebs-node-0   1/1     Running   0          111m
     pod/cassandra-openebs-node-1   1/1     Running   0          7m46s
     pod/cassandra-openebs-node-2   1/1     Running   0          6m35s

     NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
     persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-0   Bound    pvc-22b0a740-f58d-41d6-8520-325fc7266a4f   20Gi       RWO            openebs-device   4h
     persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-1   Bound    pvc-0d36674e-9094-4dca-889c-40d62bc6bb10   20Gi       RWO            openebs-device   3h59m
     persistentvolumeclaim/var-lib-cassandra-cassandra-openebs-node-2   Bound    pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827   20Gi       RWO            openebs-device   3h58m

     NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
     service/cassandra-openebs-svc   ClusterIP   10.100.9.95   <none>        7000/TCP,7001/TCP,9042/TCP   4h
    
     $ kubectl get pv
     NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                  STORAGECLASS           REASON     AGE
     pvc-0d36674e-9094-4dca-889c-40d62bc6bb10   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-1   openebs-device              8m54s
     pvc-22b0a740-f58d-41d6-8520-325fc7266a4f   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-0   openebs-device              8m58s
     pvc-a2d81f41-cb97-46f6-91c4-d0cdff2b1827   20Gi       RWO            Delete           Bound    cassandra/var-lib-cassandra-cassandra-openebs-node-2   openebs-device              8m51s

     ```
 12. Access the Database and verify the data stored previously.
    
     ```
     $ kubectl exec -it cassandra-openebs-node-0 bash -n cassandra
     cassandra@cassandra-openebs-node-0:/$ nodetool status
     Datacenter: datacenter1
     =======================
     Status=Up/Down
     |/ State=Normal/Leaving/Joining/Moving
     --  Address         Load       Tokens       Owns (effective)  Host ID                               Rack
     UN  192.168.11.219  282.12 KiB  256          34.5%             29bb331f-67a2-4440-86dc-32540aa40bc1  rack1
     UN  192.168.82.118  302.23 KiB  256          30.7%             db10a0d7-2b42-4e67-b048-b6dca0ded0a6  rack1
     UN  192.168.41.55   289.31 KiB  256          34.8%             d16694a9-18d3-4625-a6b7-346ef09545a0  rack1

     cassandra@cassandra-openebs-node-0:/$  cqlsh cassandra-openebs-svc.cassandra.svc.cluster.local
     Connected to cassandra-openebs at cassandra-openebs-svc.cassandra.svc.cluster.local:9042.
     [cqlsh 5.0.1 | Cassandra 3.11.6 | CQL spec 3.4.4 | Native protocol v4]
     Use HELP for help.
     cqlsh> use dev;
     cqlsh:dev> select * from emp;

     empid | emp_dept | emp_first | emp_last
     -------+----------+-----------+----------
         1 |      fin |      fred |    smith
         2 |    sales |       eon |   morgan
         3 |       qa |     Ashok |    Kumar

      (3 rows)
     ```
     Now, it is verified that data is same as the one before.
  

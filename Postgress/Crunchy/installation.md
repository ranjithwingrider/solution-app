1. Create a 3 Node cluster in GKE

   My setup details:

   - 1.16.13-gke-1 K8s version

   - 3 Node cluster n1-standard-2

   - 2 vCPUs/Node

   - 7.5GB RAM/Node

   - Ubuntu 18.04


2. Create 3 disks of 30G in same Zone where Nodes are created
   ```
   $ gcloud compute disks create mysql-disk1 mysql-disk2 mysql-disk3 --size=30G --zone=us-central1-c
   ```
3. Attach disk to each Node
   ```
   $ gcloud compute instances attach-disk gke-postgress-ranjith-default-pool-bafc3e6a-hln6 --disk mysql-disk1 --device-name mysql-disk1 --zone=us-central1-c

   $ gcloud compute instances attach-disk gke-postgress-ranjith-default-pool-bafc3e6a-k2w3 --disk mysql-disk2 --device-name mysql-disk2 --zone=us-central1-c

   $ gcloud compute instances attach-disk gke-postgress-ranjith-default-pool-bafc3e6a-s5l4 --disk mysql-disk3 --device-name mysql-disk3 --zone=us-central1-c
   ```
4. Connect k8s cluster with Kubera Director

5. Install OpenEBS using Kubera

6. Provision CSPC pool Kubera. I have used striped pool provisioning method.

   - Verify that CSPC has been created successfully.
     ``` 
     $  kubectl get cspc -n openebs
     NAME              HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
     cstorpool-usgqf   3                  3                      3                  48m
     ```
     ```
     $ kubectl get cspi -n openebs
     NAME                   HOSTNAME                                           ALLOCATED   FREE     CAPACITY    READONLY   PROVISIONEDREPLICAS   HEALTHYREPLICAS   TYPE     STATUS   AGE
     cstorpool-usgqf-jxbs   gke-postgress-ranjith-default-pool-bafc3e6a-s5l4   108k        28800M   28800108k   false      0                     0                 stripe   ONLINE   48m
     cstorpool-usgqf-k986   gke-postgress-ranjith-default-pool-bafc3e6a-hln6   108k        28800M   28800108k   false      0                     0                 stripe   ONLINE   48m
     cstorpool-usgqf-nskw   gke-postgress-ranjith-default-pool-bafc3e6a-k2w3   154k        28800M   28800154k   false      0                     0                 stripe   ONLINE   48m
     ```
7. Create a StorageClass and mention the following.
   
   - Name of the Storage Class
   
   - ReplicaCount of cStorVolume
   
   - CSPC pool name where the volumes are going to be created.
   
   Sample StorageClass
   ``` 
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: openebs-csi-cstor-disk
   provisioner: cstor.csi.openebs.io
   allowVolumeExpansion: true
   parameters:
     cas-type: cstor
     replicaCount: "1"
     cstorPoolCluster: cstorpool-usgqf   # Change CSPC name once CSPC pool has been created in Kubera Director
   ```
8. Apply the StorageClass
   ```
   $ kubectl apply -f sc-cspc.yaml
   storageclass.storage.k8s.io/openebs-csi-cstor-disk created
   ```
   Verify the StorageClass
   ```
   $ kubectl get sc
   NAME                        PROVISIONER                                                AGE
   openebs-csi-cstor-disk      cstor.csi.openebs.io                                       22s
   openebs-device              openebs.io/local                                           52m
   openebs-hostpath            openebs.io/local                                           52m
   openebs-jiva-default        openebs.io/provisioner-iscsi                               52m
   openebs-snapshot-promoter   volumesnapshot.external-storage.k8s.io/snapshot-promoter   52m
   standard (default)          kubernetes.io/gce-pd                                       70m
   ```

9. Get the Cass Operator manifest and update the Storage Class information wherever cStor volume need to be created.
   ```
   $ wget https://raw.githubusercontent.com/CrunchyData/postgres-operator/v4.4.1/installers/kubectl/postgres-operator.yml
   ```
   Sample manifest
   ```
   apiVersion: v1
   kind: ServiceAccount
   metadata:
       name: pgo-deployer-sa
       namespace: pgo
   ---
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: pgo-deployer-cr
   rules:
     - apiGroups:
         - ''
       resources:
         - namespaces
       verbs:
         - get
         - list
         - create
         - patch
         - delete
     - apiGroups:
         - ''
       resources:
         - pods
       verbs:
         - list
     - apiGroups:
         - ''
       resources:
         - secrets
       verbs:
         - list
         - get
         - create
         - delete
     - apiGroups:
         - ''
       resources:
         - configmaps
         - services
         - persistentvolumeclaims
       verbs:
         - get
         - create
         - delete
         - list
     - apiGroups:
         - ''
       resources:
         - serviceaccounts
       verbs:
         - get
         - create
         - delete
         - patch
         - list
     - apiGroups:
         - apps
         - extensions
       resources:
         - deployments
       verbs:
         - get
         - list
         - watch
         - create
         - delete
     - apiGroups:
         - apiextensions.k8s.io
       resources:
         - customresourcedefinitions
       verbs:
         - get
         - create
         - delete
     - apiGroups:
         - rbac.authorization.k8s.io
       resources:
         - clusterroles
         - clusterrolebindings
         - roles
         - rolebindings
       verbs:
         - get
         - create
         - delete
         - bind
         - escalate
     - apiGroups:
         - rbac.authorization.k8s.io
       resources:
         - roles
       verbs:
         - create
         - delete
     - apiGroups:
         - batch
       resources:
         - jobs
       verbs:
         - delete
         - list
     - apiGroups:
         - crunchydata.com
       resources:
         - pgclusters
         - pgreplicas
         - pgpolicies
         - pgtasks
       verbs:
         - delete
         - list
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: pgo-deployer-cm
     namespace: pgo
   data:
     values.yaml: |+
       ---
       archive_mode: "true"
       archive_timeout: "60"
       backrest_aws_s3_bucket: ""
       backrest_aws_s3_endpoint: ""
       backrest_aws_s3_key: ""
       backrest_aws_s3_region: ""
       backrest_aws_s3_secret: ""
       backrest_aws_s3_uri_style: ""
       backrest_aws_s3_verify_tls: "true"
       backrest_port: "2022"
       badger: "false"
       ccp_image_prefix: "registry.developers.crunchydata.com/crunchydata"
       ccp_image_pull_secret: ""
       ccp_image_pull_secret_manifest: ""
       ccp_image_tag: "centos7-12.4-4.4.1"
       create_rbac: "true"
       crunchy_debug: "false"
       db_name: ""
       db_password_age_days: "0"
       db_password_length: "24"
       db_port: "5432"
       db_replicas: "0"
       db_user: "testuser"
       default_instance_memory: "128Mi"
       default_pgbackrest_memory: "48Mi"
       default_pgbouncer_memory: "24Mi"
       delete_metrics_namespace: "false"
       delete_operator_namespace: "false"
       delete_watched_namespaces: "false"
       disable_auto_failover: "false"
       disable_fsgroup: "false"
       reconcile_rbac: "true"
       exporterport: "9187"
       grafana_admin_password: ""
       grafana_admin_username: "admin"
       grafana_install: "true"
       grafana_storage_access_mode: "ReadWriteOnce"
       grafana_storage_class_name: "openebs-csi-cstor-disk"
       grafana_supplemental_groups: "65534"
       grafana_volume_size: "1G"
       metrics: "false"
       metrics_namespace: "pgo"
       namespace: "pgo"
       namespace_mode: "dynamic"
       pgbadgerport: "10000"
       pgo_add_os_ca_store: "false"
       pgo_admin_password: "examplepassword"
       pgo_admin_perms: "*"
       pgo_admin_role_name: "pgoadmin"
       pgo_admin_username: "admin"
       pgo_apiserver_port: "8443"
       pgo_apiserver_url: "https://postgres-operator"
       pgo_client_cert_secret: "pgo.tls"
       pgo_client_container_install: "false"
       pgo_client_version: "4.4.1"
       pgo_cluster_admin: "false"
       pgo_disable_eventing: "false"
       pgo_disable_tls: "false"
       pgo_image_prefix: "registry.developers.crunchydata.com/crunchydata"
       pgo_image_pull_secret: ""
       pgo_image_pull_secret_manifest: ""
       pgo_image_tag: "centos7-4.4.1"
       pgo_installation_name: "devtest"
       pgo_noauth_routes: ""
       pgo_operator_namespace: "pgo"
       pgo_tls_ca_store: ""
       pgo_tls_no_verify: "false"
       pod_anti_affinity: "preferred"
       pod_anti_affinity_pgbackrest: ""
       pod_anti_affinity_pgbouncer: ""
       prometheus_install: "true"
       prometheus_storage_access_mode: "ReadWriteOnce"
       prometheus_storage_class_name: "openebs-csi-cstor-disk"
       prometheus_supplemental_groups: "65534"
       prometheus_volume_size: "1G"
       scheduler_timeout: "3600"
       service_type: "ClusterIP"
       sync_replication: "false"
       backrest_storage: "openebs"
       backup_storage: "openebs"
       primary_storage: "openebs"
       replica_storage: "openebs"
       wal_storage: ""
       storage1_name: "openebs"
       storage1_access_mode: "ReadWriteOnce"
       storage1_size: "5Gi"
       storage1_type: "dynamic"
       storage1_class: "openebs-csi-cstor-disk"
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: pgo-deployer-crb
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: pgo-deployer-cr
   subjects:
   - kind: ServiceAccount
     name: pgo-deployer-sa
     namespace: pgo
   ---
   apiVersion: batch/v1
   kind: Job
   metadata:
    name: pgo-deploy
    namespace: pgo
   spec:
    backoffLimit: 0
    template:
        metadata:
            name: pgo-deploy
        spec:
            serviceAccountName: pgo-deployer-sa
            restartPolicy: Never
            containers:
            - name: pgo-deploy
              image: registry.developers.crunchydata.com/crunchydata/pgo-deployer:centos7-4.4.1
              imagePullPolicy: IfNotPresent
              env:
                - name: DEPLOY_ACTION
                  value: install
              volumeMounts:
                - name: deployer-conf
                  mountPath: "/conf"
            volumes:
              - name: deployer-conf
                configMap:
                  name: pgo-deployer-cm
   ```
  
 10. Create a namespace `pgo`.
     ```    
     $ kubectl create namespace pgo
     ```
     Apply the above postgres-operator YAML spec
     ```  
     $ kubectl apply -f postgres-operator.yml
     serviceaccount/pgo-deployer-sa created
     clusterrole.rbac.authorization.k8s.io/pgo-deployer-cr created
     configmap/pgo-deployer-cm created
     clusterrolebinding.rbac.authorization.k8s.io/pgo-deployer-crb created
     job.batch/pgo-deploy created
     ```
     Wait for 2 mins to get the deployment for the operator has to been created.

11. Download the pg client which is needed to install pg cluster and other management operations
    ```
    curl https://raw.githubusercontent.com/CrunchyData/postgres-operator/v4.4.1/installers/kubectl/client-setup.sh > client-setup.sh
    chmod +x client-setup.sh
    ./client-setup.sh
    ```
    It will show some command at the output of above command execution which should run for setting the ENV variables and add it.
    ```
    $ cat <<EOF >> ~/.bashrc
    export PATH=/root/.pgo/pgo:$PATH
    export PGOUSER=/root/.pgo/pgo/pgouser
    export PGO_CA_CERT=/root/.pgo/pgo/client.crt
    export PGO_CLIENT_CERT=/root/.pgo/pgo/client.crt
    export PGO_CLIENT_KEY=/root/.pgo/pgo/client.key
    EOF
    ```

12. Check the Postgress default cluster status
    ```
    $pgo test -n pgo default

    cluster : default
        Services
                primary (10.96.6.3:5432): UP
        Instances
                primary (default-66b86b6b58-tgjwx): UP
     ```
     In the above result, Services and Instances status should be `UP`. Then only, you can use the database for operations.
 		
13. Check the status of PV,PVC,CVR,cStorVolume 		
    ```
    $ kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS             REASON   AGE
    pvc-18c1da1d-8b09-4e02-9bbf-9514a8fa2ee4   1Gi        RWO            Delete           Bound    pgo/default             openebs-csi-cstor-disk            7m37s
    pvc-8c4ce584-6a43-4b85-95e5-687579a28fe0   1Gi        RWO            Delete           Bound    pgo/default-pgbr-repo   openebs-csi-cstor-disk            7m37s
    
    $ kubectl get pvc -n pgo
    NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
    default             Bound    pvc-18c1da1d-8b09-4e02-9bbf-9514a8fa2ee4   1Gi        RWO            openebs-csi-cstor-disk   7m41s
    default-pgbr-repo   Bound    pvc-8c4ce584-6a43-4b85-95e5-687579a28fe0   1Gi        RWO            openebs-csi-cstor-disk   7m41s
    
    $ kubectl get cvr -n openebs
    NAME                                                            USED    ALLOCATED   STATUS    AGE
    pvc-18c1da1d-8b09-4e02-9bbf-9514a8fa2ee4-cstorpool-smkhr-rckh   46.2M   13.4M       Healthy   7m48s
    pvc-8c4ce584-6a43-4b85-95e5-687579a28fe0-cstorpool-smkhr-rckh   44.7M   11.5M       Healthy   7m48s
    
    $ kubectl get cstorvolume -n openebs
    NAME                                       STATUS    AGE     CAPACITY
    pvc-18c1da1d-8b09-4e02-9bbf-9514a8fa2ee4   Healthy   7m55s   1Gi
    pvc-8c4ce584-6a43-4b85-95e5-687579a28fe0   Healthy   7m55s   1Gi

14. Open new ssh terminal
    ```
    $ kubectl -n pgo port-forward svc/postgres-operator 8443:8443
    ```  
    
    On previous terminal
    ```	
    $ pgo version
    pgo client version 4.4.1
    pgo-apiserver version 4.4.1
    ```
    Perform the following on the previous terminal.
   
    - Create a Database Clsuter
      ```
      $ pgo create cluster -n pgo pg-database
      created cluster: pg-database
      workflow id: f151673a-6ac6-4909-ae9e-c5053a9f8e05
      database name: pg-database
      users:
          username: testuser password: L{6[L\p_LkvOog3_p7LY0\EP
      ```
      
    - Check the Postgress default cluster status
      ```
      $ pgo test -n pgo pg-database
      cluster : pg-database
      Services
         primary (10.79.6.213:5432): UP
      Instances
         primary (pg-database-6465d89cf9-q927n): UP
      ```
      In the above result, Services and Instances status should be `UP`. Then only, you can use the database for operations.
				
16. Check the status of PV,PVC,CVR,cStorVolume 	
    ```
    $ kubectl get pvc -n pgo
    NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
    pg-database             Bound    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c   5Gi        RWO            openebs-csi-cstor-disk   12m
    pg-database-pgbr-repo   Bound    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2   5Gi        RWO            openebs-csi-cstor-disk   12m

    $ kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS             REASON   AGE
    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2   5Gi        RWO            Delete           Bound    pgo/pg-database-pgbr-repo   openebs-csi-cstor-disk            12m
    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c   5Gi        RWO            Delete           Bound    pgo/pg-database             openebs-csi-cstor-disk            12m

    $ kubectl get cvr -n openebs
    NAME                                                            USED    ALLOCATED   STATUS    AGE
    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2-cstorpool-usgqf-k986   83.8M   16.5M       Healthy   12m
    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c-cstorpool-usgqf-k986   51.0M   13.6M       Healthy   12m

    $ kubectl get cv -n openebs
    NAME                                       STATUS    AGE   CAPACITY
    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2   Healthy   12m   5Gi
    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c   Healthy   12m   5Gi

    $ kubectl get pod -n openebs
    NAME                                                              READY   STATUS    RESTARTS   AGE
    cspc-operator-5ff6bf8489-dh2zp                                    1/1     Running   0          125mcstorpool-usgqf-jxbs-6565846758-czwll                             3/3     Running   1          122m
    cstorpool-usgqf-k986-764c48687f-cj722                             3/3     Running   1          122m
    cstorpool-usgqf-nskw-bf5f4bddf-lspvt                              3/3     Running   1          122m
    cvc-operator-7f94f5f9f5-hxlt6                                     1/1     Running   0          125m
    maya-apiserver-9d67ccdfd-mn4jd                                    1/1     Running   2          125m
    openebs-admission-server-747c98d589-4svft                         1/1     Running   0          125m
    openebs-cstor-admission-server-6f75477676-psk9b                   1/1     Running   0          125m
    openebs-localpv-provisioner-7495677ff8-9cvzn                      1/1     Running   0          125m
    openebs-ndm-bmccb                                                 1/1     Running   0          125m
    openebs-ndm-gw78b                                                 1/1     Running   0          125m
    openebs-ndm-operator-86449bdbb6-5bdvl                             1/1     Running   1          125m
    openebs-ndm-sfrjf                                                 1/1     Running   0          125m
    openebs-provisioner-67859b5cc9-v9hps                              1/1     Running   0          125m
    openebs-snapshot-operator-7bd666d6f5-5xjzt                        2/2     Running   0          125m
    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2-target-7d5796c48dqwkj4   3/3     Running   0          12m
    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c-target-56f9948dfdhxhzb   3/3     Running   0          12m

    $ pgo backup pg-database --backup-opts="--type=full"
    created Pgtask backrest-backup-pg-database


    $ kubectl get cvr -n openebs
    NAME                                                            USED    ALLOCATED   STATUS    AGE
    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2-cstorpool-usgqf-k986   154M    30.6M       Healthy   4h33m
    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c-cstorpool-usgqf-k986   87.8M   19.7M       Healthy   4h33m

    $ kubectl get pod -n pgo
    NAME                                                READY   STATUS      RESTARTS   AGE
    backrest-backup-pg-database-ddndz                   0/1     Completed   0          4h4mpg-database-6465d89cf9-q927n                        1/1     Running     0          4h20m
    pg-database-backrest-shared-repo-854ccf598f-gxs5z   1/1     Running     0          4h22m
    pg-database-stanza-create-29fr8                     0/1     Completed   0          4h19m
    pgo-deploy-sbp2b                                    1/1     Running     0          5h17m
    postgres-operator-58f448cd8c-lffz8                  4/4     Running     1          5h16m

    $ kubectl exec -it pg-database-6465d89cf9-q927n bash -n pgo
    bash-4.2$
    bash-4.2$
    bash-4.2$ pssql
    bash: pssql: command not found
    bash-4.2$ psql
    psql (12.4)
    Type "help" for help.

    postgres=# \l
                                   List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
    -------------+----------+----------+-------------+-------------+-----------------------
    mddb        | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
    pg-database | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres         +
                |          |          |             |             | postgres=CTc/postgres+
                |          |          |             |             | testuser=CTc/postgres
    postgres    | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
    template0   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    template1   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    (5 rows)

    postgres=#

    postgres=# \q
    bash-4.2$ pgbench -i -s 50 mddb;
    dropping old tables...
    NOTICE:  table "pgbench_accounts" does not exist, skipping
    NOTICE:  table "pgbench
    ....
    0000 of 5000000 tuples (86%) done (elapsed 43.91 s, remaining 7.15 s)
    4400000 of 5000000 tuples (88%) done (elapsed 45.37 s, remaining 6.19 s)
    4500000 of 5000000 tuples (90%) done (elapsed 45.89 s, remaining 5.10 s)
    4600000 of 5000000 tuples (92%) done (elapsed 46.69 s, remaining 4.06 s)
    4700000 of 5000000 tuples (94%) done (elapsed 46.87 s, remaining 2.99 s)
    4800000 of 5000000 tuples (96%) done (elapsed 47.71 s, remaining 1.99 s)
    4900000 of 5000000 tuples (98%) done (elapsed 49.32 s, remaining 1.01 s)
    5000000 of 5000000 tuples (100%) done (elapsed 49.46 s, remaining 0.00 s)

    vacuuming...
    creating primary keys...
    done.

17. Check the storage used in eachof the cStor Volume Replicas. 
    ``` 
    $ kubectl get cvr -n openebs
    NAME                                                            USED    ALLOCATED   STATUS    AGE
    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2-cstorpool-usgqf-k986   189M    64.0M       Healthy   5h1m
    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c-cstorpool-usgqf-k986   1.56G   381M        Healthy   5h1m

18. Check some of the database informations
    ```
    $ bash-4.2$ psql mddb;
    psql (12.4)
    Type "help" for help.
   
    mddb=# \l
                                    List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
    -------------+----------+----------+-------------+-------------+-----------------------
    mddb        | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
    pg-database | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres         +
                |          |          |             |             | postgres=CTc/postgres+
                |          |          |             |             | testuser=CTc/postgres
    postgres    | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
    template0   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    template1   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    (5 rows)
 
    mddb=# \dt
                  List of relations
    Schema |       Name       | Type  |  Owner
    --------+------------------+-------+----------
    public | pgbench_accounts | table | postgres
    public | pgbench_branches | table | postgres
    public | pgbench_history  | table | postgres
    public | pgbench_tellers  | table | postgres
    (4 rows)

    mddb=# select count(*) from pgbench_accounts;
    count
    ---------
    5000000
    (1 row)
 
19. Let's take the full backup of the database.
    ```
    postgres=# pgo backup pg-database --backup-opts="--type=full"
    postgres-#
    ```
    Get the wokflow details of the databsase.
    ```
    postgress# pgo show workflow f151673a-6ac6-4909-ae9e-c5053a9f8e05
    parameter           value
    ---------           -----
    pg-cluster          pg-database
    task completed      2020-08-27T07:15:22Z
    task submitted      2020-08-27T07:13:02Z
    workflowid          f151673a-6ac6-4909-ae9e-c5053a9f8e05

    $ pgo show cluster pg-database

    cluster : pg-database (crunchy-postgres-ha:centos7-12.4-4.4.1)
        pod : pg-database-6465d89cf9-q927n (Running) on gke-postgress-ranjith-default-pool-bafc3e6a-s5l4 (1/1) (primary)
                pvc: pg-database (5Gi)
        resources : Memory: 128Mi
        deployment : pg-database
        deployment : pg-database-backrest-shared-repo
        service : pg-database - ClusterIP (10.79.6.213)
        labels : autofail=true crunchy_collect=false deployment-name=pg-database pgo-backrest=true pgo-version=4.4.1 workflowid=f151673a-6ac6-4909-ae9e-c5053a9f8e05 crunchy-pgbadger=false crunchy-pgha-scope=pg-database name=pg-database pg-cluster=pg-database pg-pod-anti-affinity= pgouser=admin

     
    $ postgress# pgo show backup pg-database

	cluster: pg-database
	storage type: local

	stanza: db
    	status: ok
    	cipher: none

    	db (current)
        	wal archive min/max (12-1)

        	full backup: 20200827-071538F
            	timestamp start/stop: 2020-08-27 12:45:38 +0530 IST / 2020-08-27 12:46:20 +0530 IST
            	wal start/stop: 000000010000000000000002 / 000000010000000000000002
            	database size: 31.1MiB, backup size: 31.1MiB
            	repository size: 3.7MiB, repository backup size: 3.7MiB
            	backup reference list:

        	full backup: 20200827-073038F
            	timestamp start/stop: 2020-08-27 13:00:38 +0530 IST / 2020-08-27 13:01:32 +0530 IST
            	wal start/stop: 000000010000000000000005 / 000000010000000000000006
            	database size: 31.1MiB, backup size: 31.1MiB
            	repository size: 3.7MiB, repository backup size: 3.7MiB
            	backup reference list:


20. Now, Let's restore the Database cluster.	
    ```
    $ pgo create cluster restore-pg-database --restore-from=pg-database
    created cluster: restore-pg-database
    workflow id: 513ced52-2ac4-4761-9304-95de769b3760
    database name: restore-pg-database
    users:
        username: testuser password: 8jsJnO.j\NxMI,3W_6yX,AwO
    ``` 
    Verify the relevant PODs and storage volume details.
    ```	
    $ kubectl get pvc -n pgo
    NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
    pg-database             Bound    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c   5Gi        RWO            openebs-csi-cstor-disk   6h37m
    pg-database-pgbr-repo   Bound    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2   5Gi        RWO            openebs-csi-cstor-disk   6h37m
    restore-pg-database     Bound    pvc-b799145f-d6b0-4ad5-b542-fc5161af56eb   5Gi        RWO            openebs-csi-cstor-disk   47s
   
    $ kubectl get pod -n pgo
    NAME                                                READY   STATUS      RESTARTS   AGE
    backrest-backup-pg-database-4jqms                   0/1     Completed   0          6m40s
    pg-database-6465d89cf9-q927n                        1/1     Running     0          6h36m
    pg-database-backrest-shared-repo-854ccf598f-gxs5z   1/1     Running     0          6h37m
    pg-database-stanza-create-29fr8                     0/1     Completed   0          6h35m
    pgo-deploy-sbp2b                                    0/1     Error       0          7h32m
    postgres-operator-58f448cd8c-lffz8                  4/4     Running     1          7h31m
    restore-pg-database-bootstrap-lshfk                 1/1     Running     0          55s
   
    $ kubectl get pod -n pgo -o wide
    NAME                                                READY   STATUS      RESTARTS   AGE     IP           NODE                                               NOMINATED NODE   READINESS GATES
    backrest-backup-pg-database-4jqms                   0/1     Completed   0          6m46s   10.12.0.19   gke-postgress-ranjith-default-pool-bafc3e6a-s5l4   <none>           <none>
    pg-database-6465d89cf9-q927n                        1/1     Running     0          6h36m   10.12.0.17   gke-postgress-ranjith-default-pool-bafc3e6a-s5l4   <none>           <none>
    pg-database-backrest-shared-repo-854ccf598f-gxs5z   1/1     Running     0          6h37m   10.12.1.21   gke-postgress-ranjith-default-pool-bafc3e6a-hln6   <none>           <none>
    pg-database-stanza-create-29fr8                     0/1     Completed   0          6h35m   10.12.1.22   gke-postgress-ranjith-default-pool-bafc3e6a-hln6   <none>           <none>
    pgo-deploy-sbp2b                                    0/1     Error       0          7h33m   10.12.0.16   gke-postgress-ranjith-default-pool-bafc3e6a-s5l4   <none>           <none>
    postgres-operator-58f448cd8c-lffz8                  4/4     Running     1          7h31m   10.12.2.14   gke-postgress-ranjith-default-pool-bafc3e6a-k2w3   <none>           <none>
    restore-pg-database-bootstrap-lshfk                 1/1     Running     0          61s     10.12.0.20   gke-postgress-ranjith-default-pool-bafc3e6a-s5l4   <none>           <none>

    $ kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS             REASON   AGE
    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2   5Gi        RWO            Delete           Bound    pgo/pg-database-pgbr-repo   openebs-csi-cstor-disk            6h37m
    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c   5Gi        RWO            Delete           Bound    pgo/pg-database             openebs-csi-cstor-disk            6h37m
    pvc-b799145f-d6b0-4ad5-b542-fc5161af56eb   5Gi        RWO            Delete           Bound    pgo/restore-pg-database     openebs-csi-cstor-disk            75s


    $ kubectl get cvr -n openebs
    NAME                                                            USED    ALLOCATED   STATUS    AGE
    pvc-00cb0160-1b4c-4d26-8153-688534a2a3f2-cstorpool-usgqf-k986   242M    106M        Healthy   6h42m
    pvc-781a8530-eaed-48ef-b103-ca8e41f87f9c-cstorpool-usgqf-k986   1.52G   376M        Healthy   6h42m
    pvc-b799145f-d6b0-4ad5-b542-fc5161af56eb-cstorpool-usgqf-k986   865M    207M        Healthy   6m21s
    pvc-bc7c1da5-c1a4-473f-9b14-f9be53602eec-cstorpool-usgqf-nskw   89.5M   45.4M       Healthy   4m23s


    $ kubectl get pod -n pgo
    NAME                                                       READY   STATUS      RESTARTS   AGE
    backrest-backup-pg-database-4jqms                          0/1     Completed   0          12m
    backrest-backup-restore-pg-database-slqfz                  1/1     Running     0          2m45s
    pg-database-6465d89cf9-q927n                               1/1     Running     0          6h41m
    pg-database-backrest-shared-repo-854ccf598f-gxs5z          1/1     Running     0          6h43m
    pg-database-stanza-create-29fr8                            0/1     Completed   0          6h41m
    pgo-deploy-sbp2b                                           0/1     Error       0          7h38m
    postgres-operator-58f448cd8c-lffz8                         4/4     Running     1          7h37m
    restore-pg-database-78c7c4bff5-6htbq                       1/1     Running     0          3m44s
    restore-pg-database-backrest-shared-repo-f8f45b496-xzjk4   1/1     Running     0          4m48s
    restore-pg-database-bootstrap-lshfk                        0/1     Completed   0          6m46s
    restore-pg-database-stanza-create-pkqff                    0/1     Completed   0          2m49s
   
21. Verify the database details from the restored one.
    ```
    $ kubectl exec -it restore-pg-database-78c7c4bff5-6htbq bash -n pgo
    bash-4.2$ psql mddb;
    psql (12.4)
    Type "help" for help.

    mddb=# \l
                                   List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
    -------------+----------+----------+-------------+-------------+-----------------------
    mddb        | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
    pg-database | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres         +
                |          |          |             |             | postgres=CTc/postgres+
                |          |          |             |             | testuser=CTc/postgres
    postgres    | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
    template0   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    template1   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    (5 rows)

    mddb=# \dt
                List of relations
    Schema |       Name       | Type  |  Owner
    --------+------------------+-------+----------
    public | pgbench_accounts | table | postgres
    public | pgbench_branches | table | postgres
    public | pgbench_history  | table | postgres
    public | pgbench_tellers  | table | postgres
    (4 rows)

    mddb=# select count(*) from pgbench_accounts;
    count
    ---------
    5000000
    (1 row)
    ```
    From the above output, we can see that number of entires from the table is matching with the original database.




Important link
===

https://www.crunchydata.com/developers/download-postgres/containers/postgres-operator
https://access.crunchydata.com/documentation/postgres-operator/4.4.1/architecture/high-availability/
https://access.crunchydata.com/documentation/postgres-operator/4.4.1/architecture/high-availability/
https://access.crunchydata.com/documentation/postgres-operator/4.4.1/architecture/disaster-recovery/
https://access.crunchydata.com/documentation/postgres-operator/4.1.1/configuration/pgo-yaml-configuration/#miscellaneous-pgo

https://access.crunchydata.com/documentation/postgres-operator/latest/
https://access.crunchydata.com/documentation/postgres-operator/4.4.1/architecture/disaster-recovery/
https://access.crunchydata.com/documentation/postgres-operator/4.2.1/pgo-client/common-tasks/
https://access.crunchydata.com/documentation/postgres-operator/4.4.1/pgo-client/common-tasks/

Links for Commands:

https://thenewstack.io/tutorial-deploy-postgresql-on-kubernetes-running-the-openebs-storage-engine/

https://portworx.com/run-ha-postgresql-gke/

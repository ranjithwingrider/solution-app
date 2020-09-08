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
   $ gcloud compute instances attach-disk gke-postgress-openebs-default-pool-90e061ff-9x0p --disk mysql-disk1 --device-name mysql-disk1 --zone=us-central1-c

   $ gcloud compute instances attach-disk gke-postgress-openebs-default-pool-90e061ff-g2mq --disk mysql-disk2 --device-name mysql-disk2 --zone=us-central1-c

   $ gcloud compute instances attach-disk gke-postgress-openebs-default-pool-90e061ff-th7l --disk mysql-disk3 --device-name mysql-disk3 --zone=us-central1-c
   ```
4. Connect k8s cluster with Kubera Director

5. Install OpenEBS in Kubera Director

6. Provision CSPC pool in Kubera Director. I have used striped pool provisioning method.
    
   - Verify that CSPC has been created successfully.
     ``` 
     $  kubectl get cspc -n openebs
     
     NAME              HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
     cstorpool-cxfun   3                  3                      3                  25m


     ```
     Get the CSPI details.
     ```
     $ kubectl get cspi -n openebs
     
     NAME                   HOSTNAME                                     ALLOCATED   FREE     CAPACITY      READONLY   PROVISIONEDREPLICAS   HEALTHYREPLICAS   TYPE     STATUS   AGE
     cstorpool-cxfun-d84t   gke-postgre-sql-default-pool-8044c486-0blk   74k         28800M   28800074k     false      0                     0                 stripe   ONLINE   26m
     cstorpool-cxfun-kl2d   gke-postgre-sql-default-pool-8044c486-v0tz   66500       28800M   28800066500   false      0                     0                 stripe   ONLINE   26m
     cstorpool-cxfun-wm8n   gke-postgre-sql-default-pool-8044c486-6rzj   614k        28800M   28800614k     false      0                     0                 stripe   ONLINE   26m
     ```
7. Create a Storage Class and mention the following.
   
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
     cstorPoolCluster: cstorpool-cxfun   # Change CSPC name once CSPC pool has been created in Kubera Director
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
   openebs-csi-cstor-disk      cstor.csi.openebs.io                                       25s
   openebs-device              openebs.io/local                                           8m41s
   openebs-hostpath            openebs.io/local                                           8m41s
   openebs-jiva-default        openebs.io/provisioner-iscsi                               8m45s
   openebs-snapshot-promoter   volumesnapshot.external-storage.k8s.io/snapshot-promoter   8m43s
   standard (default)          kubernetes.io/gce-pd                                       43m
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
    Verify that PostgresSQL operator pod is running correctly.
    ```
    $  kubectl get pod -n pgo
    
    NAME               READY   STATUS    RESTARTS   AGE
    pgo-deploy-hvglm   1/1     Running   0          53s
    ```  
    Wait for 2 mins to create the deployment for the PostgresSQL operator has to been created.
    ```
    $ kubectl get deploy -n pgo
    
    NAME                READY   UP-TO-DATE   AVAILABLE   AGE
    postgres-operator   1/1     1            1           81s
    ```
    Verify that Postgress operator pod is running correctly.
    ```
    $  kubectl get pod -n pgo
     
    NAME                                 READY   STATUS      RESTARTS   AGE
    pgo-deploy-hvglm                     0/1     Completed   0          2m22s
    postgres-operator-58f448cd8c-xjcck   4/4     Running     1          54s
    ```
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
    
    source ~/.bashrc
    ```
12. Open a new ssh terminal
    ```
    $ kubectl -n pgo port-forward svc/postgres-operator 8443:8443
    ```  
    
    On previous terminal
    ```	
    $ pgo version
    pgo client version 4.4.1
    pgo-apiserver version 4.4.1
    ```
13. Perform the following on the previous terminal.
   
    - Create a Database Clsuter
      ```
      $ pgo create cluster -n pgo pg-database
      
      created cluster: pg-database
      workflow id: 436ba9a5-5eb6-46f4-ac19-5827f7d147bb
      database name: pg-database
      users:
            username: testuser password: /+c{\cXB2M-vfaEX2.tI91\`
      ```
      
    - Check the Postgress default cluster status
      ```
      $ pgo test -n pgo pg-database

      cluster : pg-database
            Services
                   primary (10.32.10.86:5432): UP
            Instances
                   primary (pg-database-8486fc99f4-dkfwq): UP

      ```
      In the above result, Services and Instances status should be `UP`. Then only, you can use the database for operations.
				
14. Check the status of PV,PVC,CVR,cStorVolume 	
    ```
    $ kubectl get pvc -n pgo
    NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
    pg-database             Bound    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5   5Gi        RWO            openebs-csi-cstor-disk   2m4s
    pg-database-pgbr-repo   Bound    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb   5Gi        RWO            openebs-csi-cstor-disk   2m3s


    $ kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS             REASON   AGE
    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb   5Gi        RWO            Delete           Bound    pgo/pg-database-pgbr-repo   openebs-csi-cstor-disk            2m14s
    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5   5Gi        RWO            Delete           Bound    pgo/pg-database             openebs-csi-cstor-disk            2m14s


    $ kubectl get cvr -n openebs
    NAME                                                            USED    ALLOCATED   STATUS    AGE
    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb-cstorpool-cxfun-kl2d   4.78M   66.5K       Healthy   2m25s
    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5-cstorpool-cxfun-wm8n   46.9M   12.9M       Healthy   2m26s
 

    $ kubectl get cv -n openebs
    NAME                                       STATUS    AGE    CAPACITY
    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb   Healthy   4m3s   5Gi
    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5   Healthy   4m3s   5Gi


    $ kubectl get pod -n openebs
    NAME                                                              READY   STATUS    RESTARTS   AGE
    cspc-operator-5ff6bf8489-h6pkm                                    1/1     Running   0          43m
    cstorpool-cxfun-d84t-79f78cb55d-v6tt5                             3/3     Running   2          39m
    cstorpool-cxfun-kl2d-74646f795d-tx2hx                             3/3     Running   3          39m
    cstorpool-cxfun-wm8n-6cbf775d8c-5nl54                             3/3     Running   3          39m
    cvc-operator-7f94f5f9f5-zs5fk                                     1/1     Running   0          43m
    maya-apiserver-9d67ccdfd-pccmt                                    1/1     Running   0          43m
    openebs-admission-server-747c98d589-54xzj                         1/1     Running   0          43m
    openebs-cstor-admission-server-6f75477676-nflc8                   1/1     Running   0          43m
    openebs-localpv-provisioner-7495677ff8-tr8qq                      1/1     Running   0          43m
    openebs-ndm-2n9hk                                                 1/1     Running   0          43m
    openebs-ndm-9djdd                                                 1/1     Running   0          43m
    openebs-ndm-mvlz4                                                 1/1     Running   0          43m
    openebs-ndm-operator-86449bdbb6-q4b2d                             1/1     Running   1          43m
    openebs-provisioner-67859b5cc9-wvlqb                              1/1     Running   0          43m
    openebs-snapshot-operator-7bd666d6f5-2dhfn                        2/2     Running   0          43m
    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb-target-77d986db69fgjg8   3/3     Running   0          2m43s
    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5-target-78c77c585c89cpc   3/3     Running   0          2m43s


16. Verify the status of Database status.
    ```
    $ kubectl get pod -n pgo
    
    NAME                                                READY   STATUS      RESTARTS   AGE
    backrest-backup-pg-database-7bzdb                   0/1     Completed   0          4m12s
    pg-database-8486fc99f4-dkfwq                        1/1     Running     0          5m36s
    pg-database-backrest-shared-repo-854ccf598f-2jkrl   1/1     Running     0          6m40s
    pg-database-stanza-create-cr52c                     0/1     Completed   0          4m26s
    pgo-deploy-hvglm                                    0/1     Completed   0          15m
    postgres-operator-58f448cd8c-xjcck                  4/4     Running     1          13m

    $ kubectl exec -it pg-database-8486fc99f4-dkfwq bash -n pgo

    bash-4.2$ psql
    psql (12.4)
    Type "help" for help.

    postgres=# \l
                                   List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
    -------------+----------+----------+-------------+-------------+-----------------------
    pg-database | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres         +
                |          |          |             |             | postgres=CTc/postgres+
                |          |          |             |             | testuser=CTc/postgres
    postgres    | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
    template0   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    template1   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    (4 rows)


    postgres=#

    postgres=# \q
    bash-4.2$ pgbench -i -s 50 pg-database;
    dropping old tables...
    NOTICE:  table "pgbench_accounts" does not exist, skipping
    NOTICE:  table "pgbench_branches" does not exist, skipping
    NOTICE:  table "pgbench_history" does not exist, skipping
    NOTICE:  table "pgbench_tellers" does not exist, skipping
    creating tables...
    generating data...
    100000 of 5000000 tuples (2%) done (elapsed 0.21 s, remaining 10.18 s)
    200000 of 5000000 tuples (4%) done (elapsed 0.72 s, remaining 17.32 s)
    300000 of 5000000 tuples (6%) done (elapsed 2.19 s, remaining 34.33 s)
    .......
    4400000 of 5000000 tuples (88%) done (elapsed 45.42 s, remaining 6.19 s)
    4500000 of 5000000 tuples (90%) done (elapsed 47.03 s, remaining 5.23 s)
    4600000 of 5000000 tuples (92%) done (elapsed 50.46 s, remaining 4.39 s)
    4700000 of 5000000 tuples (94%) done (elapsed 51.41 s, remaining 3.28 s)
    4800000 of 5000000 tuples (96%) done (elapsed 59.80 s, remaining 2.49 s)
    4900000 of 5000000 tuples (98%) done (elapsed 65.18 s, remaining 1.33 s)
    5000000 of 5000000 tuples (100%) done (elapsed 65.36 s, remaining 0.00 s)
    vacuuming...
    creating primary keys...
    done.
    
    bash-4.2$ psql pg-database;
    pg-database=#  \dt
                List of relations
    Schema |       Name       | Type  |  Owner
    --------+------------------+-------+----------
    public | pgbench_accounts | table | postgres
    public | pgbench_branches | table | postgres
    public | pgbench_history  | table | postgres
    public | pgbench_tellers  | table | postgres
    (4 rows)

    pg-database=# select count(*) from pgbench_accounts;
    count
    ---------
    5000000
    (1 row)
    
    pg-database=# \q
    bash-4.2$ exit
    ```
17. Check the storage used in each of the cStor Volume Replicas. 
    ``` 
    $ kubectl get cvr -n openebs
   
    NAME                                                            USED    ALLOCATED   STATUS    AGE
    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb-cstorpool-cxfun-kl2d   138M    63.4M       Healthy   15m
    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5-cstorpool-cxfun-wm8n   1.72G   421M        Healthy   15m
    ```
    
18. Let's take the full backup of the database. Login to the database and perform the backup operation. Get the wokflow details of the databsase. Get the worklflow details from Step 13. Run the command from where pgo utilitiy has installed in your local path. Ensure you have port-forwared the Postgres operator service in another shell.
    ```
    $ pgo show workflow 436ba9a5-5eb6-46f4-ac19-5827f7d147bb
    parameter           value
    ---------           -----
    task completed      2020-09-08T09:28:21Z
    task submitted      2020-09-08T09:26:06Z
    workflowid          436ba9a5-5eb6-46f4-ac19-5827f7d147bb
    pg-cluster          pg-database

    $ pgo backup pg-database --backup-opts="--type=full"
    created Pgtask backrest-backup-pg-database


    $ pgo show cluster pg-database

    cluster : pg-database (crunchy-postgres-ha:centos7-12.4-4.4.1)
        pod : pg-database-8486fc99f4-dkfwq (Running) on gke-postgre-sql-default-pool-8044c486-0blk (1/1) (primary)
                pvc: pg-database (5Gi)
        resources : Memory: 128Mi
        deployment : pg-database
        deployment : pg-database-backrest-shared-repo
        service : pg-database - ClusterIP (10.32.10.86)
        labels : crunchy-pgbadger=false pg-cluster=pg-database pgo-backrest=true pgo-version=4.4.1 workflowid=436ba9a5-5eb6-46f4-ac19-5827f7d147bb pgouser=admin autofail=true crunchy-pgha-scope=pg-database crunchy_collect=false deployment-name=pg-database name=pg-database pg-pod-anti-affinity=

     
    $ postgress# pgo show backup pg-database
    
    cluster: pg-database
    storage type: local

    stanza: db
         status: ok
         cipher: none

         db (current)
            wal archive min/max (12-1)

            full backup: 20200908-092842F
                timestamp start/stop: 2020-09-08 14:58:42 +0530 IST / 2020-09-08 14:59:22 +0530 IST
                wal start/stop: 000000010000000000000002 / 000000010000000000000002
                database size: 31.1MiB, backup size: 31.1MiB
                repository size: 3.7MiB, repository backup size: 3.7MiB
                backup reference list:

            full backup: 20200908-095052F
                timestamp start/stop: 2020-09-08 15:20:52 +0530 IST / 2020-09-08 15:22:09 +0530 IST
                wal start/stop: 000000010000000000000047 / 000000010000000000000048
                database size: 779.0MiB, backup size: 779.0MiB
                repository size: 44.3MiB, repository backup size: 44.3MiB
                backup reference list:



19. Now, Let's restore the Database cluster.	
    ```
    $ pgo create cluster restore-pg-database --restore-from=pg-database

    created cluster: restore-pg-database
    workflow id: 8bb5813f-60f6-45ca-a156-1883e2ad25f6
    database name: restore-pg-database
    users:
          username: testuser password: 8Ct3:EG<C7O3tf@S[XNyqGry
    ``` 
    Verify the relavant PODs and storage volume details.
    ```	
    $ kubectl get pvc -n pgo

    NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
    pg-database             Bound    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5   5Gi        RWO            openebs-csi-cstor-disk   30m
    pg-database-pgbr-repo   Bound    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb   5Gi        RWO            openebs-csi-cstor-disk   30m
    restore-pg-database     Bound    pvc-d1dcbaa1-ef40-4259-82bf-b1909891c9e7   5Gi        RWO            openebs-csi-cstor-disk   36s
   
    $ kubectl get pod -n pgo
    NAME                                                READY   STATUS      RESTARTS   AGE
    backrest-backup-pg-database-s6q4g                   0/1     Completed   0          6m36s
    pg-database-8486fc99f4-dkfwq                        1/1     Running     0          30m
    pg-database-backrest-shared-repo-854ccf598f-2jkrl   1/1     Running     0          31m
    pg-database-stanza-create-cr52c                     0/1     Completed   0          29m
    pgo-deploy-hvglm                                    0/1     Completed   0          39m
    postgres-operator-58f448cd8c-xjcck                  4/4     Running     1          38m
    restore-pg-database-bootstrap-vfs4p                 1/1     Running     0          57s

   
    $ kubectl get pod -n pgo -o wide
    NAME                                                READY   STATUS      RESTARTS   AGE     IP           NODE                                         NOMINATED NODE   READINESS GATES
    backrest-backup-pg-database-s6q4g                   0/1     Completed   0          7m10s   10.28.0.31   gke-postgre-sql-default-pool-8044c486-0blk   <none>           <none>
    pg-database-8486fc99f4-dkfwq                        1/1     Running     0          30m     10.28.0.29   gke-postgre-sql-default-pool-8044c486-0blk   <none>           <none>
    pg-database-backrest-shared-repo-854ccf598f-2jkrl   1/1     Running     0          31m     10.28.2.13   gke-postgre-sql-default-pool-8044c486-6rzj   <none>           <none>
    pg-database-stanza-create-cr52c                     0/1     Completed   0          29m     10.28.0.30   gke-postgre-sql-default-pool-8044c486-0blk   <none>           <none>
    pgo-deploy-hvglm                                    0/1     Completed   0          40m     10.28.2.11   gke-postgre-sql-default-pool-8044c486-6rzj   <none>           <none>
    postgres-operator-58f448cd8c-xjcck                  4/4     Running     1          38m     10.28.1.11   gke-postgre-sql-default-pool-8044c486-v0tz   <none>           <none>
    restore-pg-database-bootstrap-vfs4p                 1/1     Running     0          91s     10.28.0.32   gke-postgre-sql-default-pool-8044c486-0blk   <none>           <none>


    $ kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS             REASON   AGE
    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb   5Gi        RWO            Delete           Bound    pgo/pg-database-pgbr-repo           openebs-csi-cstor-disk            32m
    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5   5Gi        RWO            Delete           Bound    pgo/pg-database                     openebs-csi-cstor-disk            32m
    pvc-d1dcbaa1-ef40-4259-82bf-b1909891c9e7   5Gi        RWO            Delete           Bound    pgo/restore-pg-database             openebs-csi-cstor-disk            116s
    pvc-ede94f90-d19a-4901-8467-e589eecc58a3   5Gi        RWO            Delete           Bound    pgo/restore-pg-database-pgbr-repo   openebs-csi-cstor-disk            5s


    $ kubectl get cvr -n openebs
    NAME                                                            USED    ALLOCATED   STATUS    AGE
    pvc-61bcdc37-5ccb-4cfa-b38b-ae3de9ada6bb-cstorpool-cxfun-kl2d   246M    114M        Healthy   33m
    pvc-bba40176-4ca5-4f1c-a8e4-db3c3b65b0e5-cstorpool-cxfun-wm8n   1.64G   409M        Healthy   33m
    pvc-d1dcbaa1-ef40-4259-82bf-b1909891c9e7-cstorpool-cxfun-wm8n   847M    203M        Healthy   2m50s
    pvc-ede94f90-d19a-4901-8467-e589eecc58a3-cstorpool-cxfun-wm8n   3.53M   56.5K       Healthy   59s


    $ kubectl get pod -n pgo
    NAME                                                       READY   STATUS      RESTARTS   AGE
    backrest-backup-pg-database-s6q4g                          0/1     Completed   0          9m9s
    pg-database-8486fc99f4-dkfwq                               1/1     Running     0          32m
    pg-database-backrest-shared-repo-854ccf598f-2jkrl          1/1     Running     0          33m
    pg-database-stanza-create-cr52c                            0/1     Completed   0          31m
    pgo-deploy-hvglm                                           0/1     Completed   0          42m
    postgres-operator-58f448cd8c-xjcck                         4/4     Running     1          40m
    restore-pg-database-7f587b5f84-pb62h                       0/1     Running     0          51s
    restore-pg-database-backrest-shared-repo-f8f45b496-57p2c   1/1     Running     0          99s
    restore-pg-database-bootstrap-vfs4p                        0/1     Completed   0          3m30s

   
20. Verify the database details from the restored one.
    ```
    $ kubectl exec -it restore-pg-database-7f587b5f84-pb62h bash -n pgo
    bash-4.2$ psql pg-database;
    psql (12.4)
    Type "help" for help.

    pg-database=# \l
                                   List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
    -------------+----------+----------+-------------+-------------+-----------------------
    pg-database | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres         +
                |          |          |             |             | postgres=CTc/postgres+
                |          |          |             |             | testuser=CTc/postgres
    postgres    | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
    template0   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    template1   | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
                |          |          |             |             | postgres=CTc/postgres
    (4 rows)

    pg-database=# \dt
                 List of relations
    Schema |       Name       | Type  |  Owner
    --------+------------------+-------+----------
    public | pgbench_accounts | table | postgres
    public | pgbench_branches | table | postgres
    public | pgbench_history  | table | postgres
    public | pgbench_tellers  | table | postgres
    (4 rows)


    pg-database=# select count(*) from pgbench_accounts;
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

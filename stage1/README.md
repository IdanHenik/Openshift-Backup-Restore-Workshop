# Stage 1: Backup and Restore MongoDB using OADP in OpenShift

In this stage, you will learn how to backup and restore a MongoDB database in an OpenShift cluster using the OADP (OpenShift Application Data Protection) Operator. MongoDB is a popular NoSQL database, and it's essential to know how to perform data protection tasks in a containerized environment like OpenShift.

## Prerequisites

Before you begin, ensure you have the following prerequisites in place:

- Access to an OpenShift cluster with cluster-admin premissions.
- Acces to a S3 Bucket.
- The OADP Operator installed on your OpenShift cluster. If not installed, follow the official documentation to [install the OADP Operator](https://github.com/kastenhq/Documentation).

## Step-by-Step Guide

### 1. Deploy a MongoDB StatefulSet

1.1. Create a New Project: `mongodb/mongodb-statefulset.yaml`.
```bash
oc new-project mongodb-demo
```

1.2. Add Helm Repo and create environment vars:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   ```
   ```bash
   export MONGODB_ROOT_PASSWORD=redhat123
   export MONGODB_REPLICA_SET_KEY=redhat123
   ```
1.3  Install the Helm Repo by using helm install, make sure to set required SCC to mongodb 'serviceAccount':
```bash
 mongodb bitnami/mongodb --set podSecurityContext.fsGroup="",containerSecurityContext.runAsUser="1001080001",podSecurityContext.enabled=false,architecture=replicaset,auth.replicaSetKey=$MONGODB_REPLICA_SET_KEY,auth.rootPassword=$MONGODB_ROOT_PASSWORD,volumePermissions.enabled=true
```
1.4 Create a MongoDB Client to verify the connection:
```bash
kubectl/oc run --namespace mongodb-demo mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:4.4.13-debian-10-r9 --command -- bash
```
1.5 Verify connection to the DB:
```bash
mongo "mongodb://mongodb-0.mongodb-headless.mongodb-demo.svc.cluster.local:27017,mongodb-1.mongodb-headless.mongodb-demo.svc.cluster.local:27017" --authenticationDatabase admin  -u root -p $MONGODB_ROOT_PASSWORD
```
1.6 Create a database and create a document in post collection:
```bash
use demo-db

db.post.insert([
  {
    title: "MongoDB Backup DEMO",
    description: "Demo DB",
    by: "ihenik",
    url: "http://redhat.com",
    tags: ["mongodb", "demo", "BACKUP"],
  }
])
```
1.7 Verify the document is created:
```bash
db.post.find()
```
![db-post](images/db-post.jpg)
### 2. OpenShift API for Data Protection Operator


2.1 You can install the OADP Operator from the Openshift's OperatorHub. You can search for the operator using keywords such as 'oadp' or 'velero'.

![OADP](images/OADP-operatorhub.jpg)

Now click on 'Install'

Click on 'Install' again. This will create Project openshift-adp if it does not exist, and install the OADP operator in it.

2.2 Create Credentails secret for OADP operator to use.

```bash
oc create secret generic cloud-credentials --namespace openshift-adp --from-file cloud=secret.yaml
```
secret.yaml:

```yaml
[default]
aws_access_key_id=<INSERT_VALUE>
aws_secret_access_key=<INSERT_VALUE>
```
2.3 Create DataProtectionApplication (DPA) CR [DataProtectionApplication.yaml](https://github.com/IdanHenik/Openshift-Backup-Restore-Workshop/DataProtectionApplication.yaml).:
```bash
oc apply -f DataProtectionApplication.yaml
```
DataProtectionApplication.yaml:

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa
  namespace: openshift-adp
spec:
  backupLocations:
    - velero:
        config:
          profile: default
          region: eu-west-1
        credential:
          key: cloud
          name: cloud-credentials #Secret name we created earlier
        default: true
        objectStorage:
          bucket: backup-demo-ihenik #BucketName
          prefix: stage1 #PrefixName
        provider: aws
  configuration:
    restic:
      enable: true
    velero:
      defaultPlugins:
        - openshift
        - aws
        - csi
  snapshotLocations:
    - velero:
        config:
          profile: default
          region: eu-west-1
        provider: aws
```
![DPA](images/DPA.jpg)

NOTE: you can use AWS/ODF/Minio or another provider velero support [linkforvelero]

2.4 Modify VolumeSnapShotClass

The Velero CSI plugin, to backup CSI backed PVCs, will choose the VolumeSnapshotClass in the cluster that has the same driver name and also has the velero.io/csi-volumesnapshot-class: "true" label set on it.

```bash
oc patch volumesnapshotclass <volumesnapshotclass-name> --type=merge -p '{"deletionPolicy": "Retain"}'
oc label volumesnapshotclass <volumesnapshotclass-name> velero.io/csi-volumesnapshot-class="true"
```
![CSI](images/CSI.jpg)

### 3.Backup Application
At this step, you have to create a Backup CR which will backup the required application.
```bash
oc apply -f Backup.yaml
```
Backup.yaml:

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  annotations:
    velero.io/source-cluster-k8s-gitversion: v1.25.12+ba5cc25
    velero.io/source-cluster-k8s-major-version: '1'
    velero.io/source-cluster-k8s-minor-version: '25'
  name: backup-demo
  namespace: openshift-adp
  labels:
    velero.io/storage-location: dpa-1 #Location created by DPA's CR
spec:
  csiSnapshotTimeout: 10m0s
  defaultVolumesToFsBackup: false
  includedNamespaces: #Namespace it will backup
    - mongodb-demo
  itemOperationTimeout: 1h0m0s
  storageLocation: dpa-1
  ttl: 720h0m0s
  volumeSnapshotLocations:
    - dpa-1
```
The status of backup should eventually show Phase: 'Completed'
![BK](images/BK.jpg)

### Disaster :(
Let's assume something horrible happend and somehow the Entire Namespace got deleted.

[Image of Deletion]

![Deletion](images/ns-del2.jpg)
![Deletion2](images/ns-delete.jpg)

### Restore The Application
You have been called to fix the problem and you remembered you preformed a backup.

In order to restore the backup you have to create a 'Restore' CR:
```bash
oc apply -f Restore.yaml
```
Restore.yaml:

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore
  namespace: openshift-adp
spec:
  backupName: backup-demo #Name of the backup you created eariler
  excludedResources:
    - nodes
    - events
    - events.events.k8s.io
    - backups.velero.io
    - restores.velero.io
    - resticrepositories.velero.io
    - csinodes.storage.k8s.io
    - volumeattachments.storage.k8s.io
    - backuprepositories.velero.io
  includedNamespaces:
    - mongodb-demo #Restore mentioned NS
  itemOperationTimeout: 1h0m0s
  restorePVs: true #Restore the PVs as well


```

The status of restore should eventually show Phase: 'Completed'.
![ns-restore](images/db-restore.jpg)

After a few minutes, you should see the chat application up and running.
![db-restore](images/db-restore.jpg)
![post-restore](images/post-restore.jpg)
Good job !










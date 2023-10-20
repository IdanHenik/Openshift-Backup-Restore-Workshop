# Stage 2: Backup and Restore RocketChat between OpenShift Clusters using OADP

In this stage, you will learn how to backup and restore a RocketChat application between two OpenShift clusters using the OADP (OpenShift Application Data Protection) Operator. This scenario demonstrates the importance of data protection and disaster recovery in a multi-cluster environment.

## Prerequisites

Before you begin, ensure you have the following prerequisites in place:

- Access to two OpenShift clusters with an admin premissions: a source cluster and a target cluster.
- The OADP Operator installed on both clusters.
- DataProtectionApplication CR is created and BackupStorageLocation is available at the Primary cluster.
- S3 access.

## Scenario I

At this scenario we will backup manually the application from the primary cluster and restore it at the passive cluster.

### 1. Deploy RocketChat on the Source Cluster


1.1. Apply the YAML file to deploy the RocketChat application on the source cluster:
   ```bash
   git clone && cd stage2
   oc apply -f manifests/manifest.yaml
   ```
1.2 Setup the application.
   ```bash
   oc get route rocket-chat -n rocket-chat -ojsonpath="{.spec.host}"
   ```
   Insert the URL and keep with the setup guidelines.

   a

  Hit on 'Enter' to continue and keep a head with the guidelines.


### 2.Backup RocketChat from the Source Cluster
2.1 At this step, you have to create a Backup CR which will backup the required application.
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
  name: backup-demo-rocket-chat
  namespace: openshift-adp
  labels:
    velero.io/storage-location: dpa-1 #Location created by DPA's CR
spec:
  csiSnapshotTimeout: 10m0s
  defaultVolumesToFsBackup: true
  includedNamespaces: #Namespace it will backup
    - rocket-chat
  itemOperationTimeout: 1h0m0s
  storageLocation: dpa-1
  ttl: 720h0m0s
  volumeSnapshotLocations:
    - dpa-1
```
### 3. Configure the passive cluster
3.1 Install OADP - you can use the guidelines from stage 1 [Stage 1: Backup and Restore MongoDB using OADP](stage1/stage1.md)

3.2 Create Credentails secret for OADP operator to use.

```bash
oc create secret generic cloud-credentials --namespace openshift-adp --from-file cloud=secret.yaml
```
secret.yaml:

```yaml
[default]
aws_access_key_id=<INSERT_VALUE>
aws_secret_access_key=<INSERT_VALUE>
```
3.3 Create DataProtectionApplication (DPA) CR:
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
          prefix: backup-demo #PrefixName
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
3.4 Modify VolumeSnapShotClass

The Velero CSI plugin, to backup CSI backed PVCs, will choose the VolumeSnapshotClass in the cluster that has the same driver name and also has the velero.io/csi-volumesnapshot-class: "true" label set on it.

```bash
oc patch volumesnapshotclass <volumesnapshotclass-name> --type=merge -p '{"deletionPolicy": "Retain"}'
oc label volumesnapshotclass <volumesnapshotclass-name> velero.io/csi-volumesnapshot-class="true"
```
Great, now that all set we should be able to see under 'Backup' tab the backup we created at step 2.

![workspace](images/backup-show.png)

### 4. Preform a Restore at the passive cluster
4.1 You know what to do now, In order to restore the backup you have to create a 'Restore' CR:
```bash
oc apply -f Restore.yaml
```
Restore.yaml:

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-rocket-chat
  namespace: openshift-adp
spec:
  backupName: backup-demo-rocket-chat #Name of the backup you created eariler
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
    - rocket-chat #Restore mentioned NS
  itemOperationTimeout: 1h0m0s
  restorePVs: true #Restore the PVs as well
```
When you navigate to the application namespace (rocket-chat), you will encounter the following situation: 
![workspace](images/mess.png)

But why does this occur? When OADP executes a restore flow, the 'restore-controller' follows an 'existingUpdatePolicy' as the default behavior. If the controller detects that the object already exists, it will skip the operation. In this case, we should modify the policy value to 'update', which instructs the controller to update the existing resources.

But there's more to it. As you probably know, an application might require specific pre-configuration tasks to function properly. 
That's why it's essential to characterize each backup carefully.

In our example, the MongoDB instance must initialize a replica set named "rs0" with a single member. To meet this requirement, we can utilize 'restore-hooks.' These hooks will execute the specified command after the restore process has been completed inside a designated container.

4.2 Now after we learn few things let's deploy the Restore in the right way:
```bash
oc apply -f Restore.yaml
```
Restore.yaml:

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-rocket-chat
  namespace: openshift-adp
spec:
  hooks:
    resources:
      - includedNamespaces:
          - rocket-chat
        labelSelector:
          matchExpressions:
            - key: posthook
              operator: Exists
        postHooks:
          - exec:
              command:
                - /bin/sh
                - '-c'
                - >-
                  sleep 60 && mongo rocket-chat-db:27017 --eval
                  "rs.initiate({_id: 'rs0', members: [{_id:0,
                  host:'localhost:27017'}]})"
              execTimeout: 1m
              waitTimeout: 5m
              onError: Fail
              container: rocketchat-db # which contianer to execute the hook on
        name: restore-hook-1
  backupName: backup-demo-rocket-chat
  existingResourcePolicy: update # if mentioned it will update resources and not skip if exsist
  restorePVs: true # allow to restore pvs
```
Now if you navigate to the app's routes in each cluster you need to see the following state:
![workspace](images/same-start.png)

Excellent ! but what happen when the application continues to write more data to the volume ? Manually executing this entire procedure can be quite daunting...

Let's see how to deal with that in the next scenriao
## Scenario II
As mentioned earlier, applications naturally generate new data, modify existing data, and manually managing backups may not be the most efficient approach. This is particularly true in the context of an active-passive architecture where we definitely don't want to restore each component manually. So, how do we address this? In this scenario, we implement a 'scheduled' backup and restore strategy.

First Let's create some data by creating a new channel and some messeages:
![workspace](images/channel.png)
![workspace](images/mesg.png)

Now we have two un-synced applications:

![workspace](images/unsync.png)

### 1. At the primary cluster create a Schedule CR
```bash
oc apply -f Schedule.yaml
```
Schedule.yaml:

```yaml
  apiVersion: velero.io/v1
  kind: Schedule
  metadata:
    name: rocket-chat
    namespace: openshift-adp
  spec:
    schedule: '*/20 * * * * ' #schedule time
    template:
      defaultVolumesToFS: true
      hooks: {}
      includedNamespaces:
        - rocket-chat
      storageLocation: dpa-1
      ttl: 720h0m0s #backup-expried time
```
After creating this CR a backup will created schedually by the time-period you mentioned.

![workspace](images/scheduled-backup.png)

### 2. Creating a scheduled Restore at the passive-cluster

Unfortunately, OADP isn't equipped to schedule a restore, but don't fret, because good old cron jobs come to the rescue! :)
2.1 Create a crobjob to restore schedullay the newest backup:
```bash
oc apply -f CronJob.yaml
```
CronJob.yaml:
```yaml
kind: CronJob
apiVersion: batch/v1
metadata:
  name: rocket-chat-updater
  namespace: openshift-adp
spec:
  schedule: '*/25 * * * *'
  concurrencyPolicy: Allow
  suspend: false
  jobTemplate:
    metadata:
      creationTimestamp: null
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
            - name: restore-schedule
              image: bitnami/kubectl
              command:
              - "/bin/sh"
              - "-c"
              - |
                # Capture the newest backup name
                NEWEST_BACKUP_NAME=$(kubectl get backups -n <your-namespace> --sort-by=.metadata.creationTimestamp -o json | jq -r '.items[-1].metadata.name')

                # Create the Velero Restore CR with the captured backup name as the backupName
                cat <<EOF | kubectl apply -f -
                apiVersion: velero.io/v1
                kind: Restore
                metadata:
                  name: restore-rocket-chat
                  namespace: openshift-adp
                spec:
                  backupName: $NEWEST_BACKUP_NAME
                  # Rest of your Velero Restore CR specification
                   hooks:
                      resources:
                        - includedNamespaces:
                            - rocket-chat
                          labelSelector:
                            matchExpressions:
                              - key: posthook
                                operator: Exists
                          postHooks:
                            - exec:
                                command:
                                  - /bin/sh
                                  - '-c'
                                  - >-
                                    sleep 60 && mongo rocket-chat-db:27017 --eval
                                    "rs.initiate({_id: 'rs0', members: [{_id:0,
                                    host:'localhost:27017'}]})"
                                execTimeout: 1m
                                waitTimeout: 5m
                                onError: Fail
                                container: rocketchat-db # which contianer to execute the hook on
                          name: restore-hook-
                    existingResourcePolicy: update # if mentioned it will update resources and not skip if exsist
                    restorePVs: true # allow to restore pvs
                EOF
              resources: {}
              terminationMessagePath: /dev/termination-log
              terminationMessagePolicy: File
              imagePullPolicy: Always
          restartPolicy: OnFailure
          terminationGracePeriodSeconds: 30
          dnsPolicy: ClusterFirst
          serviceAccountName: velero
          serviceAccount: velero #use velero serviceaccount or create you own
          securityContext: {}
          schedulerName: default-scheduler
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1

```
If a disaster will happend, you're are now ready for it !
Every 25 mintues a new restore will be triggred, the restore will trigger a post-hook and as result we will recive an active-passive sync between the apps.
![workspace](images/synced.png)

Note: In production environments, we configure a Global Load Balancer between the applications to ensure a seamless experience for our customers.

So far, you've successfully completed basic stateful sets backups and restores, both locally and across clusters. You've also implemented a scheduling mechanism to achieve active-passive synchronization. If you're ready to take the scenario to the next level and fully backup and restore clusters and cluster hubs, proceed to the next stage :)

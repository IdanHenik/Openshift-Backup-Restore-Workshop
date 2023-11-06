# OpenShift OADP Workshop

Welcome to the OpenShift OADP Workshop! This workshop is designed to provide you with hands-on experience in backing up and restoring applications in OpenShift using various tools and techniques.

## Workshop Overview

This workshop is divided into three stages, each focusing on different aspects of application backup and restore methods using OADP on top of OpenShift. Please follow the links below to access each stage:

1. [Stage 1: Backup and Restore MongoDB using OADP](stage1/README.md)
2. [Stage 2: Backup and Restore RocketChat between OpenShift Clusters using OADP](stage2/README.md)
3. [Stage 3: Backup and Restore between Management Hub Clusters using ACM, RHGitops Operator, and OADP](stage3/README.md)

Before you begin, make sure you have the necessary prerequisites and access to OpenShift clusters. Each stage includes detailed step-by-step instructions and sample configurations to help you learn and practice the concepts.

## Prerequisites

- Access to OpenShift clusters (for each stage).
- Familiarity with Kubernetes and OpenShift concepts.

## Additional Resources

- [OpenShift Documentation](https://docs.openshift.com)
- [OADP Operator Documentation](https://docs.openshift.com/container-platform/4.12/backup_and_restore/application_backup_and_restore/oadp-intro.html)
- [ACM Documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/)
- [RHGitops Operator Documentation](https://docs.openshift.com/gitops/1.10/understanding_openshift_gitops/about-redhat-openshift-gitops.html)

## Additional Information

### What is OADP

OADP is the OpenShift API for Data Protection operator. This open source operator sets up and installs Velero on the OpenShift platform, allowing users to backup and restore applications.

### Backup Workflow 


    1. OADP using The Velero client makes a call to the Kubernetes API server to create a Backup object.

    2. The BackupController notices the new Backup object and performs validation.

    3. The BackupController begins the backup process. It collects the data to back up by querying the API server for resources.

    4. The BackupController makes a call to the object storage service – for example, AWS S3 – to upload the backup file.

![backup-process](images/backup-process.png)

### Restore Workflow



    1. The Velero client makes a call to the Kubernetes API server to create a Restore object.

    2. The RestoreController notices the new Restore object and performs validation.

    3. The RestoreController fetches the backup information from the object storage service. It then runs some preprocessing on the backed up resources to make sure the resources will work on the new cluster. For example, using the backed-up API versions to verify that the restore resource will work on the target cluster.

    4. The RestoreController starts the restore process, restoring each eligible resource one at a time.

### Object Storage Sync

Object Storage Synchronization

Velero regards object storage as the authoritative data source, regularly verifying the presence of the correct backup resources. In cases where a properly formatted backup file exists in the storage bucket, yet there is no corresponding backup resource in the Kubernetes API, Velero ensures synchronization by transferring the information from object storage to Kubernetes

### Backup file format

A backup is a compressed tar file in gzip format, and its filename corresponds to the metadata.name defined during the creation of the Backup Custom Resource.

When utilizing cloud object storage, each backup file resides within its own subdirectory located in the bucket specified in the Velero server configuration. This subdirectory contains an additional file named 'velero-backup.json.' This JSON file encompasses all pertinent details about the associated Backup resource, including any default settings, providing a comprehensive historical record of the backup configuration. The JSON file also specifies the 'status.version,' which corresponds to the format of the output file.





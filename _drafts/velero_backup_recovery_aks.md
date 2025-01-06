---
layout: post
title: Velero for Backup and Recovery
image: "https://onthedock.github.io/images/velero.jpg"
category: [AKS, Microservices, DisasterRecovery]
description: Velero for backup and recovery of AKS services? Why does AKS services need to be backedup? I'll review a use-case I recently implemented velero.
---

# Background
Velero (formerly Heptio Ark) gives you tools to back up and restore your Kubernetes cluster resources and persistent volumes. You can run Velero with a cloud provider or on-premises. Velero lets you:

    Take backups of your cluster and restore in case of loss.
    Migrate cluster resources to other clusters.
    Replicate your production cluster to development and testing clusters.

Upon first thought, one might think aks requires such a such a solution after all isnt AKS designed to be fault tolerant? 
Although, thats true AKS lacks is data persistance the underlyning persistant volumes in D/R or app specific failures arent addressed.
This is where solutions like Velero come in. Velero is used in the enterprise for backups and D/R, custom health monitoring, and even has failover.
# Pre-req for AKS
- Make sure kubectl config and your context is set to the cluster (ideally clusteradmin or rolebinding enough for velero to create definitions and pods)
  - Havent connected to your primary AKS cluster yet to run velero yet? STOP. Do this first!!

- Velero will need this to create a new namespace create a deployment in this new namespce
# Installation
There are few components required
Velero client Installation
At the time of this post, version v1.15.1 was the most current

## Client Installation (linux) 
[For windows see microsofts docs](https://learn.microsoft.com/en-us/azure/aks/aksarc/backup-workload-cluster#install-velero-with-azure-blob-storage)

```
curl -OL https://github.com/vmware-tanzu/velero/releases/download/v1.15.1/velero-v1.15.1-linux-amd64.tar.gz
```

## Installation of velero server-side components
### Storage Account

### Velero Install command
Using the client its possible to install valero as follows
```
velero install --provider azure --plugins velero/velero-plugin-for-microsoft-azure:v1.5.0 --bucket velero --
secret-file ./velero.txt --backup-location-config resourceGroup=rg_kafka,storageAccount=kafkastorageoldap,subscriptionId=8661b24d-af2f-4b2d-a05e-6fcf3b8601f2 --use-node-agent
```
> Note: You will find in microsoft documentation --use-restic. This is depricated its now been changed to --use-node-agent. use-node-agent is important to have as it instructs velero to backup aks storage like perstitent volumes. [@refer velero 7123 issue](https://github.com/vmware-tanzu/velero/issues/7123)

After installation 
```
kubectl -n velero get pods
NAME                     READY   STATUS    RESTARTS   AGE
node-agent-2l52t         1/1     Running   0          3m47s
velero-dcc6dd686-9jvbf   1/1     Running   0          13m
```

# Backup an AKS cluster

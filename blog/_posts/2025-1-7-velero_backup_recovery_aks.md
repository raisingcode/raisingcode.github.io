---
layout: post
author: "Jason Phillips"
title: "Velero for Backup and Recovery"
image: "https://onthedock.github.io/images/velero.jpg"
category: [Azure, Microservices, DisasterRecovery]
description: The post discusses the use Velero for AKS D/R
---

# Velero for Backup and Recovery
Velero gives you tools to back up and restore your Kubernetes cluster resources and persistent volumes. You can run Velero with a cloud provider or on-premises. 

## But why...
Upon first thought, one might think aks shouldn't require such a pet-like solution after all isnt AKS designed to be fault tolerant? 
Although, thats has the abilility to rebalance pods load and scale resources on Nodes etc. AKS lacksdata persistance the underlyning persistant volumes in D/R or app specific failures arent addressed major infrastructure envents. What about regional failures of the underlying Single region AKS? 
This is where solutions like Velero come in.

Velero is used in the enterprise for backups and D/R, custom health monitoring, and even has failover.

## Flexibility
Velero can backup by namespace, multiple namespaces, exclude CRDs, and backup specific by a label annotation.
For us this was ideal, since we have other services in the same namespace that we could potentially filter out

## Use-Case
 Applications anything from from wordpress to an advance microservice could be be good potential use case for velero. 
 
Working on self-hosted Kafka project, we needed to preserve the LDAP (users and objects.) so verlero was a very quick and feasible D/R solution.

## Pre-req for AKS
- Make sure kubectl config is set for primary and secondary aks cluster (ideally clusteradmin or rolebinding enough for velero to create definitions and pods)
  - Havent connected to your primary AKS cluster yet to run velero yet? STOP. Do this first!!
  - Switch first the primary cluster if you have not

- Velero will need this to create a new namespace create a deployment in this new namespce
## Installation
There are few components required
Velero client Installation
At the time of this post, version v1.15.1 was the most current

## Client Installation (linux) 
[For windows see microsofts docs](https://learn.microsoft.com/en-us/azure/aks/aksarc/backup-workload-cluster#install-velero-with-azure-blob-storage)

```
curl -OL https://github.com/vmware-tanzu/velero/releases/download/v1.15.1/velero-v1.15.1-linux-amd64.tar.gz
```

### Create the Azure Resources
*It is assumed AKS cluster and applicaton is already configured in the enviornment*

1. Create a Storage Account and blob container
``` 
az login 
az storage account create --name <StorageAccountName> --resource-group <Resourceroup> --sku Standard_GRS --encryption-services blob --https-only true --kind BlobStorage --access-tier Hot

# replace storeaccountname and resource group
# For the container <blob_container> I used velero

az storage container create -n <blob_container> --public-access --account-name <name of storageaccount>

```
2. Create a Service Principal

*We will need to store the values of the secret and the sp appId for velero*
```
AZURE_CLIENT_SECRET=`az ad sp create-for-rbac --name "velero" --role "Contributor" --query 'password' -o tsv --scopes  /subscriptions/$AZURE_SUBSCRIPTION_ID`

AZURE_CLIENT_ID=`az ad sp list --display-name "velero" --query '[0].appId' -o tsv`

```
 
## Installation of velero server-side components
The client will install velero on server with the following command

1. Install Velero Agents on Primary/Secondary AKS

Before we begin we must create velero.txt file that will be used to pass information to velero agents 

*velero.txt*
```
echo AZURE_SUBSCRIPTION_ID=<your subscription id storage account>
echo AZURE_CLIENT_ID=$AZURE_CLIENT_ID >> velero.txt
echo AZURE_CLIENT_SECRET=$AZURE_CLIENT_SECRET >> velero.txt
echo AZURE_RESOURCE_GROUP=<your resource group name for velero>
echo AZURE_CLOUD_NAME=AzurePublicCloud

```


### Velero Install command
Create velero agent on the primary and secondary aks clusters by running this command
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

## Backuping an microservice namespace
```
velero backup create fullnamespacebk --include-namespaces [app namespace]
```

## More Granular Backups
It is possible can assign data annotion to resources for finegrained backup and recovery of recoures
Example openldap service

```
kubectl label statefulset ldap backup=true -n [namespace]
kubectl label service ldap-service backup=true -n [namespace]
kubectl label configmap customldif backup=true -n [namespace]
kubectl label configmap ldap-ldifs backup=true -n [namespace]
 kubectl label secret sslcerts-volume backup=true -n [namespace]

velero backup create <your backup name>   --include-namespaces=kafka-public   --include-resources=statefulsets,services,persistentvolumeclaims,persistentvolumes,configmaps,secrets   --selector=backup=true
```

## Restoring to secondary
On the secondary/fail-over cluster (switch over with kubectl config use-context)

```
velero restore create --from-backup <backup name>
```

## Final Thoughts
This really touches the surfaces. Its possible to easily set up a schedules minutes, hours, days, etc for a point time recovery.

Velero is a power tool for D/R of you AKS CRD (container resources) and persistent volumes allowing for recovery of microservice state in DR. Since 1.15 there has been several changes included the removal of restic so be cautious when reviewing microsoft docs. 

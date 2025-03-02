---
layout: post
title: "Lessons Learned from Deploying Confluent Kafka on AKS"
date: 2025-03-01
categories: [kubernetes, kafka, azure]
tags: [aks, kafka, confluent, kubernetes]
image: /assets/img/kafka-spash.jpg
---

# Lessons Learned from Deploying Confluent Kafka on AKS

Deploying Confluent Kafka on Azure Kubernetes Service (AKS) offers powerful capabilities for event streaming, but comes with unique challenges. In this blog post, I'll share practical insights from a recent deployment, covering common issues and their solutions.

## Container Registry Configuration

One of the first hurdles was configuring our Azure Container Registry (ACR) correctly.

### Importing Confluent Images

When working with Confluent images, you'll need to import them into your ACR:

```bash
set ACR_NAME=your-acr-name

# Import key Confluent components with specific versions
az acr import --name %ACR_NAME% --source docker.io/confluentinc/cp-kafka --image cp-kafka:7.7.2
az acr import --name %ACR_NAME% --source docker.io/confluentinc/cp-server --image cp-server:7.7.2
az acr import --name %ACR_NAME% --source docker.io/confluentinc/cp-server-connect --image cp-server-connect:7.7.2
az acr import --name %ACR_NAME% --source docker.io/confluentinc/cp-ksqldb-server --image cp-ksqldb-server:7.7.2
az acr import --name %ACR_NAME% --source docker.io/confluentinc/cp-schema-registry --image cp-schema-registry:7.7.2
az acr import --name %ACR_NAME% --source docker.io/confluentinc/cp-kafka-rest --image cp-kafka-rest:7.7.2
az acr import --name %ACR_NAME% --source docker.io/confluentinc/confluent-operator --image confluent-operator:0.1145.6
az acr import --name %ACR_NAME% --source docker.io/confluentinc/confluent-init-container --image confluent-init-container:2.9.4
```

To avoid Docker Hub rate limits, authenticate first:

```bash
set DOCKER_USER=your-username
set DOCKER_PAT=your-pat

az acr import --name %ACR_NAME% --source docker.io/confluentinc/cp-kafka --image cp-kafka:7.7.2 --username %DOCKER_USER% --password %DOCKER_PAT%
```

## Confluent for Kubernetes Deployment

### Helm Installation

Deploying Confluent for Kubernetes (CFK) requires special attention to image registry and versioning:

```bash
helm upgrade --install confluent-operator .\confluent-for-kubernetes -n kafkapreprod \
  --set image.registry=your-acr.azurecr.io \
  --set global.registry.fqdn=your-acr.azurecr.io \
  --set image.repository=confluent-operator \
  --set global.initContainer.repository=confluent-init-container \
  --set image.tag="0.1145.6" \
  --set kafka.image.tag="7.7.2" \
  --set connect.image.tag="7.7.2" \
  --set controlcenter.image.tag="7.7.2" \
  --set ksqldb.image.tag="7.7.2" \
  --set schemaregistry.image.tag="7.7.2" \
  --set licenseKey="your-license-key-here"
```

When using a local ACR, you need to remove the `confluentinc/` repository prefix and specify each component's image tag.

## Persistent Volume Management

AKS persistent volume management can be tricky with Kafka:

### Stuck Persistent Volumes

If you ever need to force delete stuck PVs:

```powershell
# First change Retain policy to Delete for relevant PVs
$retainPVs = @(
    "pvc-cb30adbe-ab1b-4deb-bdd7-61996cf00244",
    "pvc-ccf4f715-c799-4a69-bcd7-29560b9de23b"
)

foreach ($pv in $retainPVs) {
    kubectl patch pv $pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
}

# Then remove finalizers and force delete
kubectl patch pv $pvName -p '{"metadata":{"finalizers":null}}' --type=merge
kubectl delete pv $pvName --force --grace-period=0
```

Remember that `helm delete` intentionally doesn't delete PVs to protect your data.

## SSL/TLS Configuration

Configuring Kafka with SSL requires careful certificate handling:

### P12 Certificate Conversion

When using P12 certificates:

```powershell
# Convert P12 to PEM
openssl pkcs12 -in certificate.p12 -out certificate.pem -nokeys
```

For Windows and older OpenSSL versions, you might need:

```bash
openssl pkcs12 -legacy -in truststore.p12 -out truststore.pem -nokeys
```

## Role-Based Access Control

RBAC is critical for secure Kafka operations:

### Schema Registry Permissions

When Schema Registry shows authorization errors:

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: ConfluentRoleBinding
metadata:
  name: sr-schemas-owner
  namespace: kafkapreprod
spec:
  principal:
    name: "User:sr"
    type: User
  role: ResourceOwner
  resourcePatterns:
    - name: "_schemas" 
      patternType: LITERAL
      resourceType: Topic
    - name: "_schemaregistry_kafkapreprod"
      patternType: LITERAL
      resourceType: Topic
  kafkaRestClassRef:
    name: default
```

## Troubleshooting Common Issues

### INCONSISTENT_CLUSTER_ID Error

This typically occurs when Kafka brokers have different cluster IDs:

```
[ERROR] org.apache.kafka.raft.KafkaRaftClient handleUnexpectedError - [RaftManager id=0] 
Unexpected error INCONSISTENT_CLUSTER_ID in FETCH response
```

Resolution requires cleaning up Kafka resources and redeploying:

```powershell
# Delete Kafka pods
kubectl delete pod kafka-0 kafka-1 kafka-2 -n kafkapreprod --force --grace-period=0

# Delete Kafka PVCs
kubectl delete pvc data0-kafka-0 data0-kafka-1 data0-kafka-2 -n kafkapreprod --force

# Restart Kafka deployment
kubectl rollout restart statefulset kafka -n kafkapreprod
```

If you need to preserve a specific cluster ID, ensure it's set consistently across redeployments.

### Metric Reporter TLS Issues

When encountering errors about metric reporter TLS:

```yaml
metricReporter:
  enabled: true
  tls:
    enabled: true
    jksPassword:
      secretRef: kafka-metric-secret
      key: password
```

## Azure Storage Integration

For tiered storage with Azure Blob:

```yaml
server:
  - confluent.tier.feature=true
  - confluent.tier.enable=true
  - confluent.tier.backend=AzureBlockBlob
  - confluent.tier.azure.block.blob.container=broker-sandbox
  - confluent.tier.azure.block.blob.cred.file.path=/mnt/secrets/credential/blob-cred.json
```

Remember to create and mount the Azure credentials secret properly.

## Conclusion

Deploying Confluent Kafka on AKS provides a powerful and scalable event streaming platform, but self-hosting Kafka on AKS can be daunting. I can personally attest to this - my first setup of Confluent Kafka was an exercise in patience and perseverance. What seemed straightforward in documentation became a labyrinth of interdependent configurations, cryptic error messages, and late nights troubleshooting unexpected behaviors. The learning curve was steep, and at times I questioned if self-hosting was the right approach at all.

While there are challenges around storage management, authentication, cluster configuration, and maintaining high availability, understanding these common issues and their solutions will help ensure a successful deployment.

For production environments, I strongly recommend implementing proper backup and recovery tooling like Velero. Velero can help you:
- Back up and restore your entire Kafka cluster, including all CRDs and state
- Create scheduled backups for disaster recovery
- Migrate cluster resources between environments
- Protect against accidental deletions or corruptions

Remember to maintain backups of critical data, especially when modifying persistent volumes, and always follow security best practices for credential management. Proper disaster recovery planning will save you from many of the headaches discussed in this post - headaches I've experienced firsthand.

Given the complexity involved, organizations should also consider Confluent Cloud as an alternative to self-managed deployments, particularly for production workloads where reliability is critical.

Happy streaming!

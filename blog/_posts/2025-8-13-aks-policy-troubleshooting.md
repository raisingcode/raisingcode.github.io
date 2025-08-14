---
layout: post
title: "Troubleshooting AKS Azure Policy Conflicts with Kubernetes Ingress"
categories: jekyll update
image: "/assets/img/aks_policy.png"
---

# Understanding AKS Azure Policy: A Practical Guide (Part 1 - The Basics)

*Published: August 2025 | 10 min read*

You've just been told to enable CIS benchmarks on your AKS clusters. The security team is happy. Compliance is checked off. Then your deployments start failing with cryptic error messages about "denied by azurepolicy-k8srequiredhttps" and you have no idea why.

This two-part series will help you understand and work with AKS policies. Part 1 covers the fundamentals - what these policies are, how they work, and basic troubleshooting. Part 2 will dive into advanced scenarios and real-world workarounds.

## What Are AKS Azure Policies?

Azure Policy for AKS is Microsoft's way of enforcing governance and compliance rules on your Kubernetes clusters. Think of it as guardrails that prevent non-compliant resources from being created.

### The Policy Ecosystem

<div align="center">
<svg width="800" height="150" xmlns="http://www.w3.org/2000/svg">
  <!-- Background -->
  <rect width="800" height="150" fill="#f8f9fa" stroke="#e9ecef" stroke-width="1"/>
  
  <!-- Boxes -->
  <rect x="10" y="50" width="120" height="50" fill="#007bff" stroke="#0056b3" stroke-width="2" rx="5"/>
  <text x="70" y="70" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Azure Policy</text>
  <text x="70" y="85" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Service</text>
  
  <rect x="170" y="50" width="120" height="50" fill="#17a2b8" stroke="#117a8b" stroke-width="2" rx="5"/>
  <text x="230" y="70" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Policy</text>
  <text x="230" y="85" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Initiatives</text>
  
  <rect x="330" y="50" width="120" height="50" fill="#28a745" stroke="#1e7e34" stroke-width="2" rx="5"/>
  <text x="390" y="70" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Individual</text>
  <text x="390" y="85" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Policies</text>
  
  <rect x="490" y="50" width="120" height="50" fill="#ffc107" stroke="#d39e00" stroke-width="2" rx="5"/>
  <text x="550" y="70" text-anchor="middle" fill="black" font-family="Arial, sans-serif" font-size="12">AKS</text>
  <text x="550" y="85" text-anchor="middle" fill="black" font-family="Arial, sans-serif" font-size="12">Cluster</text>
  
  <rect x="650" y="10" width="120" height="50" fill="#dc3545" stroke="#bd2130" stroke-width="2" rx="5"/>
  <text x="710" y="30" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">OPA</text>
  <text x="710" y="45" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Gatekeeper</text>
  
  <rect x="650" y="90" width="120" height="50" fill="#6c757d" stroke="#545b62" stroke-width="2" rx="5"/>
  <text x="710" y="110" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Admission</text>
  <text x="710" y="125" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="12">Webhooks</text>
  
  <!-- Arrows -->
  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#333" />
    </marker>
  </defs>
  
  <line x1="130" y1="75" x2="170" y2="75" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
  <text x="150" y="70" text-anchor="middle" fill="#333" font-family="Arial, sans-serif" font-size="10">Assigns</text>
  
  <line x1="290" y1="75" x2="330" y2="75" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
  <text x="310" y="70" text-anchor="middle" fill="#333" font-family="Arial, sans-serif" font-size="10">Contains</text>
  
  <line x1="450" y1="75" x2="490" y2="75" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
  <text x="470" y="70" text-anchor="middle" fill="#333" font-family="Arial, sans-serif" font-size="10">Syncs to</text>
  
  <line x1="610" y1="75" x2="640" y2="50" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
  <text x="615" y="55" text-anchor="middle" fill="#333" font-family="Arial, sans-serif" font-size="10">Creates</text>
  
  <line x1="710" y1="60" x2="710" y2="90" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
  <text x="750" y="75" text-anchor="middle" fill="#333" font-family="Arial, sans-serif" font-size="10">Enforces via</text>
</svg>
</div>

<br/>

### Common Policy Initiatives

Most organizations don't pick individual policies. They enable entire initiatives:

| Initiative | Number of Policies | Common Use Case |
|-----------|-------------------|-----------------|
| **CIS Microsoft Azure Foundations Benchmark** | 200+ | General compliance |
| **Azure Security Benchmark** | 150+ | Microsoft's recommended security baseline |
| **ISO 27001:2013** | 100+ | International security standard |
| **PCI-DSS** | 50+ | Payment card industry |
| **HIPAA HITRUST** | 80+ | Healthcare compliance |

## How Policies Actually Work in AKS

### Step 1: Policy Assignment

When you assign a policy to your AKS cluster:

```bash
# Example: Assigning CIS Benchmark
az policy assignment create \
  --name "CIS-Benchmark-AKS" \
  --scope "/subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.ContainerService/managedClusters/{cluster-name}" \
  --policy-set-definition "/providers/Microsoft.Authorization/policySetDefinitions/cis-benchmark-v1.3.0"
```

### Step 2: The Translation Layer

Azure Policy doesn't directly touch your cluster. Instead:

1. **Azure Policy** defines rules in Azure Resource Manager
2. **Policy Add-on** translates these to Kubernetes constraints
3. **OPA Gatekeeper** enforces them in your cluster

### Step 3: Enforcement via Admission Control

Here's what happens when you deploy a resource:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  ports:
  - port: 8080  # Non-standard port
    targetPort: 8080
```

The flow:
1. `kubectl apply` sends this to the API server
2. API server calls admission webhooks
3. Gatekeeper webhook evaluates against policies
4. If non-compliant: **Request denied**
5. If compliant: Resource created

## Your First Policy Encounter

Let's walk through a typical first experience with AKS policies:

### The Deployment That Worked Yesterday

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-app
  template:
    metadata:
      labels:
        app: simple-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
```

### The Error After Enabling Policies

```bash
$ kubectl apply -f simple-app.yaml
error validating data: admission webhook "validation.gatekeeper.sh" denied the request: 
[azurepolicy-k8srequiredlabels] Resource is missing required labels: {"env", "version"}
[azurepolicy-k8snolatestimage] Container 'app' is using latest tag
[azurepolicy-k8srequiredresources] Container 'app' is missing resource limits
```

### Understanding the Errors

Each error follows a pattern:
- **[azurepolicy-{constraint}]**: The policy that blocked you
- **Clear description**: What's wrong
- **Specific details**: Which container/field is affected

## Basic Troubleshooting Toolkit

### 1. See What Policies Are Active

```bash
# List all constraints in your cluster
kubectl get constraints

# Example output:
NAME                                              ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
azurepolicy-k8srequiredlabels                    deny                142
azurepolicy-k8snolatestimage                     deny                38
azurepolicy-k8srequiredresources                 deny                201
azurepolicy-k8srequiredhttpsonly                 deny                17
```

### 2. Get Policy Details

```bash
# See what a specific policy requires
kubectl describe k8srequiredlabels azurepolicy-k8srequiredlabels

# Look for the 'Spec' section:
Spec:
  Match:
    Kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet", "DaemonSet"]
  Parameters:
    labels: ["env", "app", "version"]
```

### 3. Check Recent Violations

```bash
# See recent admission webhook denials
kubectl get events --all-namespaces \
  --field-selector reason=FailedAdmission \
  --sort-by='.lastTimestamp'
```

### 4. Test Before Deploying

```bash
# Use dry-run to test without actually creating
kubectl apply -f my-manifest.yaml --dry-run=server

# If it passes dry-run, it will pass actual deployment
```

## Common Starter Policies and Quick Fixes

Here are the policies you'll hit first and how to fix them:

### 1. Required Labels

**Policy**: `k8srequiredlabels`  
**Error**: "Resource is missing required labels"

```yaml
# Before (fails)
metadata:
  name: my-app
  
# After (passes)
metadata:
  name: my-app
  labels:
    app: my-app
    env: production
    version: "1.0.0"
```

### 2. No Latest Image Tag

**Policy**: `k8snolatestimage`  
**Error**: "Container is using latest tag"

```yaml
# Before (fails)
containers:
- name: app
  image: nginx:latest
  
# After (passes)
containers:
- name: app
  image: nginx:1.25.3
```

### 3. Resource Limits Required

**Policy**: `k8srequiredresources`  
**Error**: "Container is missing resource limits"

```yaml
# Before (fails)
containers:
- name: app
  image: nginx:1.25.3
  
# After (passes)
containers:
- name: app
  image: nginx:1.25.3
  resources:
    limits:
      cpu: "500m"
      memory: "512Mi"
    requests:
      cpu: "100m"
      memory: "128Mi"
```

### 4. HTTPS Services Only

**Policy**: `k8srequiredhttpsonly`  
**Error**: "Service port name does not start with 'https'"

```yaml
# Before (fails)
spec:
  ports:
  - name: web
    port: 443
    
# After (passes)
spec:
  ports:
  - name: https-web
    port: 443
```

## Understanding Enforcement Modes

Policies can run in different modes:

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Enforce** | Blocks non-compliant resources | Production protection |
| **Audit** | Logs violations but allows creation | Testing impact |
| **Disabled** | Policy not evaluated | Temporary relief |

### Checking Policy Mode

```bash
# See enforcement action for all constraints
kubectl get constraints -o wide

# Check specific policy mode
kubectl get k8srequiredlabels azurepolicy-k8srequiredlabels -o jsonpath='{.spec.enforcementAction}'
```

### Changing Policy Mode (Temporary)

```bash
# Switch to audit mode for testing
kubectl patch k8srequiredlabels azurepolicy-k8srequiredlabels \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/enforcementAction", "value": "dryrun"}]'
```

⚠️ **Warning**: This change is temporary. Azure Policy will sync and revert it within 15 minutes.

## Creating Your First Compliant Deployment

Let's build a deployment that passes common policies:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compliant-app
  labels:
    app: compliant-app
    env: production
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: compliant-app
  template:
    metadata:
      labels:
        app: compliant-app
        env: production
        version: "1.0.0"
    spec:
      # Security context for the pod
      securityContext:
        runAsNonRoot: true
        fsGroup: 65534
      
      # Don't automount service account token
      automountServiceAccountToken: false
      
      containers:
      - name: app
        image: nginx:1.25.3  # Specific version, not latest
        
        # Container security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsUser: 65534
          capabilities:
            drop:
            - ALL
        
        # Resource limits and requests
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "128Mi"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        
        ports:
        - containerPort: 8080
          name: https-web  # Name starts with https
        
        # Mount required directories as emptyDir
        volumeMounts:
        - name: var-cache-nginx
          mountPath: /var/cache/nginx
        - name: var-run
          mountPath: /var/run
      
      volumes:
      - name: var-cache-nginx
        emptyDir: {}
      - name: var-run
        emptyDir: {}
```

## Quick Reference: Policy to Solution Mapping

| If you see this error... | Add this to your manifest... |
|-------------------------|------------------------------|
| "missing required labels" | `metadata.labels: {app: "name", env: "prod", version: "1.0"}` |
| "using latest tag" | Use specific version: `image: nginx:1.25.3` |
| "missing resource limits" | Add resources section with limits and requests |
| "automounting service account token" | `automountServiceAccountToken: false` |
| "running as root" | `securityContext.runAsNonRoot: true` |
| "no liveness probe" | Add livenessProbe configuration |
| "no readiness probe" | Add readinessProbe configuration |

## What's Next?

In Part 2, we'll tackle:
- Advanced policies that break real applications (Kafka, databases, operators)
- Policy exemptions and when to use them
- Custom policies for your organization
- Automated compliance with mutation webhooks
- GitOps strategies for policy compliance

## Key Takeaways

1. **Policies are preventive, not reactive** - They block at creation time
2. **Start with audit mode** - See what would break before enforcing
3. **Understand the requirements** - Use `kubectl describe` on constraints
4. **Test with dry-run** - Validate before deploying
5. **Build compliant templates** - Start with good defaults

Remember: These policies exist to protect your cluster. The goal is to work with them, not against them. Once you understand the basics, you can make informed decisions about when to comply and when to seek exemptions.

---

*Stay tuned for Part 2 coming soon! We'll dive into advanced scenarios including enterprise software challenges (Confluent Kafka, databases, operators), policy exemption strategies, and real-world workarounds for when policies clash with production requirements.*
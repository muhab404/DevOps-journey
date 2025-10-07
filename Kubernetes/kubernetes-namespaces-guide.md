# 🏢 Kubernetes Namespaces - DevOps Interview Guide

## Table of Contents
1. [Namespaces Overview](#namespaces-overview)
2. [Default Namespaces](#default-namespaces)
3. [Creating and Managing Namespaces](#creating-and-managing-namespaces)
4. [Resource Isolation](#resource-isolation)
5. [Cross-Namespace Communication](#cross-namespace-communication)
6. [Resource Quotas and Limits](#resource-quotas-and-limits)
7. [RBAC and Security](#rbac-and-security)
8. [Practical Examples](#practical-examples)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Interview Questions](#interview-questions)

---

## 🔹 Namespaces Overview

### What are Kubernetes Namespaces?

**Namespaces** provide a mechanism for isolating groups of resources within a single cluster, enabling multi-tenancy and resource organization.

### Key Concepts

```yaml
# Namespace Benefits:
Resource Isolation:
  - Logical separation of resources
  - Prevent naming conflicts
  - Organize resources by environment/team

Security Boundaries:
  - RBAC policies per namespace
  - Network policies for isolation
  - Service account separation

Resource Management:
  - Resource quotas per namespace
  - Limit ranges for defaults
  - Cost allocation and tracking
```

### Namespace Scope

```yaml
# Namespaced Resources:
- Pods, Services, Deployments
- ConfigMaps, Secrets
- PersistentVolumeClaims
- ServiceAccounts, Roles, RoleBindings
- NetworkPolicies, ResourceQuotas

# Cluster-Scoped Resources (Not Namespaced):
- Nodes, PersistentVolumes
- ClusterRoles, ClusterRoleBindings
- StorageClasses, CustomResourceDefinitions
- Namespaces themselves
```

---

## 🔹 Default Namespaces

### System Namespaces

```bash
# List all namespaces
kubectl get namespaces

# Default system namespaces:
default         # Default namespace for resources
kube-system     # Kubernetes system components
kube-public     # Publicly readable resources
kube-node-lease # Node heartbeat information (1.14+)
```

### Default Namespace

```yaml
# Resources created without namespace go to 'default'
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  # No namespace specified = goes to 'default'
spec:
  containers:
  - name: app
    image: nginx:1.21
```

### Kube-System Namespace

```bash
# View system components
kubectl get pods -n kube-system

# Common system components:
# - kube-apiserver
# - kube-controller-manager
# - kube-scheduler
# - etcd
# - kube-proxy
# - CoreDNS
# - CNI plugins
```

---

## 🔹 Creating and Managing Namespaces

### Imperative Creation

```bash
# Create namespace
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace development

# Create with labels
kubectl create namespace production --labels=environment=prod,team=platform

# Delete namespace (deletes all resources in it)
kubectl delete namespace staging
```

### Declarative Creation

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
    cost-center: engineering
  annotations:
    description: "Production environment for web applications"
    contact: "platform-team@company.com"
    created-by: "devops-team"
spec:
  # Namespace finalizers (optional)
  finalizers:
  - kubernetes
```

### Advanced Namespace Configuration

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    security-level: high
    compliance: pci-dss
    environment: production
  annotations:
    scheduler.alpha.kubernetes.io/node-selector: "security=high"
    description: "High-security namespace for PCI-DSS compliant workloads"
spec:
  finalizers:
  - kubernetes
---
# Namespace with resource quotas
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: secure-namespace
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "10"
    persistentvolumeclaims: "20"
---
# Namespace with limit ranges
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: secure-namespace
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2000m"
      memory: "4Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    type: Container
```

### Namespace Operations

```bash
# Set default namespace for kubectl
kubectl config set-context --current --namespace=production

# View current namespace
kubectl config view --minify | grep namespace

# Create resource in specific namespace
kubectl create deployment nginx --image=nginx:1.21 -n production

# Get resources from specific namespace
kubectl get pods -n production
kubectl get all -n production

# Get resources from all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Describe namespace
kubectl describe namespace production
```

---

## 🔹 Resource Isolation

### DNS and Service Discovery

```yaml
# Service DNS Names:
# <service-name>.<namespace>.svc.cluster.local

# Same namespace (short form):
http://web-service:80

# Different namespace (full form):
http://web-service.production.svc.cluster.local:80

# Cross-namespace service access:
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: data-tier
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  namespace: web-tier
spec:
  containers:
  - name: app
    image: web-app:v1.0
    env:
    - name: DB_HOST
      value: "database-service.data-tier.svc.cluster.local"
    - name: DB_PORT
      value: "5432"
```

### Network Isolation

```yaml
# Default: All namespaces can communicate
# Use NetworkPolicies for isolation

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}  # Apply to all pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
---
# Allow specific cross-namespace communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-web-tier
  namespace: data-tier
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: web-tier
    - podSelector:
        matchLabels:
          app: web-app
    ports:
    - protocol: TCP
      port: 5432
```

---

## 🔹 Cross-Namespace Communication

### Service Communication

```yaml
# Web tier namespace
apiVersion: v1
kind: Namespace
metadata:
  name: web-tier
  labels:
    tier: web
---
# Data tier namespace
apiVersion: v1
kind: Namespace
metadata:
  name: data-tier
  labels:
    tier: data
---
# Database service in data-tier
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: data-tier
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# Web app accessing database across namespaces
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: web-tier
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: web-app:v1.0
        env:
        - name: DATABASE_URL
          value: "postgresql://user:pass@postgres-service.data-tier.svc.cluster.local:5432/mydb"
```

### ExternalName Services

```yaml
# Create alias for external service
apiVersion: v1
kind: Service
metadata:
  name: external-api
  namespace: production
spec:
  type: ExternalName
  externalName: api.external-service.com
  ports:
  - port: 443
    targetPort: 443
---
# Use external service from different namespace
apiVersion: v1
kind: Pod
metadata:
  name: client-pod
  namespace: development
spec:
  containers:
  - name: client
    image: curl:latest
    command: ["curl"]
    args: ["https://external-api.production.svc.cluster.local/api/v1/data"]
```

---

## 🔹 Resource Quotas and Limits

### Comprehensive Resource Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    
    # Storage resources
    requests.storage: "1Ti"
    persistentvolumeclaims: "50"
    
    # Object counts
    pods: "100"
    services: "20"
    secrets: "50"
    configmaps: "50"
    
    # Specific resource types
    count/deployments.apps: "30"
    count/services.loadbalancers: "5"
    count/services.nodeports: "10"
    
    # Extended resources
    requests.nvidia.com/gpu: "10"
  
  # Scope selectors
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority", "medium-priority"]
```

### Priority-Based Quotas

```yaml
# High priority quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority"]
---
# Best effort quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: best-effort-quota
  namespace: production
spec:
  hard:
    pods: "20"
  scopes: ["BestEffort"]
```

### Limit Ranges

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: namespace-limits
  namespace: production
spec:
  limits:
  # Container limits
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
      ephemeral-storage: "1Gi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
      ephemeral-storage: "100Mi"
    max:
      cpu: "4000m"
      memory: "8Gi"
      ephemeral-storage: "10Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
      ephemeral-storage: "50Mi"
    maxLimitRequestRatio:
      cpu: "4"
      memory: "2"
  
  # Pod limits
  - type: Pod
    max:
      cpu: "8000m"
      memory: "16Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
  
  # PVC limits
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"
    min:
      storage: "1Gi"
```

---

## 🔹 RBAC and Security

### Namespace-Scoped RBAC

```yaml
# ServiceAccount for namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
---
# Role with namespace permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-manager
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
# Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-pod-manager
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
- kind: User
  name: "developer@company.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

### Cross-Namespace RBAC

```yaml
# ClusterRole for cross-namespace access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-reader
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
  resourceNames: ["public-api"]
---
# Bind cluster role to user
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cross-namespace-access
subjects:
- kind: User
  name: "platform-admin@company.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-reader
  apiGroup: rbac.authorization.k8s.io
```

### Pod Security Standards

```yaml
# Namespace with Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: secure-apps
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## 🔹 Practical Examples

### 1. Multi-Environment Setup

```yaml
# Development Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
    team: engineering
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
---
# Staging Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    team: engineering
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    pods: "50"
---
# Production Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: prod
    team: engineering
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
```

### 2. Team-Based Isolation

```yaml
# Frontend Team Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-team
  labels:
    team: frontend
    department: engineering
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: frontend-team
  name: frontend-developer
rules:
- apiGroups: ["", "apps", "extensions"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-binding
  namespace: frontend-team
subjects:
- kind: Group
  name: "frontend-developers"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: frontend-developer
  apiGroup: rbac.authorization.k8s.io
---
# Backend Team Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: backend-team
  labels:
    team: backend
    department: engineering
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: backend-team
  name: backend-developer
rules:
- apiGroups: ["", "apps", "extensions"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-team-binding
  namespace: backend-team
subjects:
- kind: Group
  name: "backend-developers"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: backend-developer
  apiGroup: rbac.authorization.k8s.io
```

### 3. Microservices Architecture

```yaml
# API Gateway Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: api-gateway
  labels:
    tier: gateway
    component: ingress
---
# User Service Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: user-service
  labels:
    tier: application
    service: user
---
# Order Service Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: order-service
  labels:
    tier: application
    service: order
---
# Database Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: database
  labels:
    tier: data
    component: storage
---
# Cross-service communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-gateway
  namespace: user-service
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: gateway
    ports:
    - protocol: TCP
      port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-services-to-db
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: application
    ports:
    - protocol: TCP
      port: 5432
```

### 4. Monitoring and Logging Namespace

```yaml
# Monitoring Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    component: monitoring
    managed-by: platform-team
---
# Cluster-wide monitoring permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-binding
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔹 Best Practices

### 1. Namespace Design

```yaml
# Naming Conventions:
Environment-based:
  - development, staging, production
  - dev-frontend, prod-backend

Team-based:
  - frontend-team, backend-team, data-team
  - platform-engineering, security-team

Service-based:
  - user-service, order-service, payment-service
  - api-gateway, database, monitoring

# Labeling Strategy:
Standard Labels:
  - environment: dev/staging/prod
  - team: frontend/backend/platform
  - component: api/database/cache
  - version: v1.0.0
  - managed-by: helm/kustomize
```

### 2. Resource Management

```yaml
# Always set resource quotas
# Use limit ranges for defaults
# Monitor resource usage
# Plan for growth and scaling

# Resource Quota Guidelines:
Development: 25% of cluster resources
Staging: 25% of cluster resources  
Production: 50% of cluster resources
```

### 3. Security Considerations

```yaml
# Security Best Practices:
- Use least privilege RBAC
- Implement network policies
- Enable Pod Security Standards
- Regular security audits
- Monitor cross-namespace access

# Network Isolation:
- Default deny all traffic
- Explicit allow rules
- Monitor network flows
- Regular policy reviews
```

### 4. Operational Excellence

```yaml
# Monitoring and Alerting:
- Resource quota utilization
- Namespace creation/deletion
- Cross-namespace communication
- RBAC violations
- Resource limit breaches

# Documentation:
- Namespace purpose and ownership
- Resource allocation rationale
- Security policies and exceptions
- Troubleshooting procedures
```

---

## 🔹 Troubleshooting

### Common Issues

#### 1. Resource Quota Exceeded

```bash
# Check quota status
kubectl describe quota -n production

# Check current usage
kubectl top pods -n production

# Common solutions:
# - Increase quota limits
# - Optimize resource requests
# - Clean up unused resources
```

#### 2. Cross-Namespace Communication Issues

```bash
# Test DNS resolution
kubectl run test-pod --image=busybox -it --rm -- nslookup service-name.namespace.svc.cluster.local

# Check network policies
kubectl get networkpolicy -n target-namespace
kubectl describe networkpolicy policy-name -n target-namespace

# Test connectivity
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://service.namespace.svc.cluster.local
```

#### 3. RBAC Permission Issues

```bash
# Check permissions
kubectl auth can-i create pods --as=user@company.com -n production

# Check role bindings
kubectl get rolebindings -n production
kubectl describe rolebinding binding-name -n production

# Check service account
kubectl get serviceaccount -n production
kubectl describe serviceaccount sa-name -n production
```

### Debugging Commands

```bash
# Namespace operations
kubectl get namespaces
kubectl describe namespace production

# Resource quotas and limits
kubectl get quota -A
kubectl get limitrange -A

# RBAC debugging
kubectl get roles,rolebindings -n production
kubectl auth can-i --list --as=user@company.com -n production

# Network policies
kubectl get networkpolicy -A
kubectl describe networkpolicy policy-name -n namespace

# Cross-namespace resources
kubectl get services -A | grep external
kubectl get endpoints -A
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: What are Kubernetes namespaces and why are they important?**
- Logical isolation of resources within a cluster
- Enable multi-tenancy and resource organization
- Provide security boundaries and resource management

**Q2: What resources are namespaced vs cluster-scoped?**
- **Namespaced**: Pods, Services, Deployments, ConfigMaps, Secrets, PVCs
- **Cluster-scoped**: Nodes, PVs, ClusterRoles, StorageClasses, Namespaces

**Q3: How do services communicate across namespaces?**
- Use FQDN: `service-name.namespace.svc.cluster.local`
- Network policies control cross-namespace traffic
- RBAC controls access permissions

### Technical Questions

**Q4: How do you implement resource quotas in namespaces?**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    pods: "50"
```

**Q5: What happens when you delete a namespace?**
- All resources in the namespace are deleted
- Finalizers may prevent immediate deletion
- PVs with Retain policy are preserved
- External resources may need manual cleanup

**Q6: How do you set default resource limits for a namespace?**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
```

### Scenario-Based Questions

**Q7: Design namespace strategy for a multi-team organization.**
```yaml
# Environment-based namespaces per team
# Team-specific RBAC policies
# Resource quotas based on team needs
# Network policies for isolation
# Monitoring and cost allocation
```

**Q8: How would you migrate resources between namespaces?**
```bash
# Export resource without namespace-specific fields
kubectl get deployment app -o yaml --export > app.yaml
# Edit namespace in YAML
# Apply to new namespace
kubectl apply -f app.yaml -n new-namespace
```

**Q9: Troubleshoot cross-namespace service communication failure.**
```bash
# Check DNS resolution
# Verify network policies
# Test connectivity
# Check RBAC permissions
# Validate service endpoints
```

### Best Practices Questions

**Q10: What are namespace management best practices?**
- Use consistent naming conventions
- Implement resource quotas and limits
- Apply appropriate RBAC policies
- Use network policies for isolation
- Monitor resource usage and costs
- Document namespace purposes and ownership
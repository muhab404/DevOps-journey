# 🔐 Kubernetes RBAC - DevOps Interview Guide

## Table of Contents
1. [RBAC Overview](#rbac-overview)
2. [Core Components](#core-components)
3. [ServiceAccounts](#serviceaccounts)
4. [Roles & ClusterRoles](#roles--clusterroles)
5. [RoleBindings & ClusterRoleBindings](#rolebindings--clusterrolebindings)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Interview Questions](#interview-questions)

---

## 🔹 RBAC Overview

### What is RBAC?

**Role-Based Access Control (RBAC)** is a method of regulating access to Kubernetes resources based on the roles of individual users or service accounts.

**Key Principles:**
- **Least Privilege**: Grant minimum permissions needed
- **Separation of Duties**: Different roles for different responsibilities
- **Defense in Depth**: Multiple layers of security controls

### RBAC Components

```yaml
# RBAC Flow:
User/ServiceAccount → RoleBinding → Role → Resources
                   ↘ ClusterRoleBinding → ClusterRole → Cluster Resources
```

### Authentication vs Authorization

```yaml
# Authentication: "Who are you?"
- X.509 certificates
- Bearer tokens
- Service account tokens
- OpenID Connect (OIDC)

# Authorization: "What can you do?"
- RBAC (Role-Based Access Control)
- ABAC (Attribute-Based Access Control)
- Node authorization
- Webhook authorization
```

---

## 🔹 Core Components

### RBAC API Objects

| Object | Scope | Purpose |
|--------|-------|---------|
| **Role** | Namespace | Define permissions within namespace |
| **ClusterRole** | Cluster | Define cluster-wide permissions |
| **RoleBinding** | Namespace | Bind Role to subjects in namespace |
| **ClusterRoleBinding** | Cluster | Bind ClusterRole to subjects cluster-wide |
| **ServiceAccount** | Namespace | Identity for pods and processes |

### Subject Types

```yaml
# 1. User (human users)
subjects:
- kind: User
  name: "alice@company.com"
  apiGroup: rbac.authorization.k8s.io

# 2. Group (collection of users)
subjects:
- kind: Group
  name: "developers"
  apiGroup: rbac.authorization.k8s.io

# 3. ServiceAccount (pod identity)
subjects:
- kind: ServiceAccount
  name: "my-service-account"
  namespace: "default"
```

---

## 🔹 ServiceAccounts

### ServiceAccount Basics

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: production
  labels:
    app: web-application
    component: backend
  annotations:
    description: "Service account for web application backend"
automountServiceAccountToken: true  # Default: true
```

### ServiceAccount with Secrets

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: registry-sa
  namespace: default
secrets:
- name: registry-secret
imagePullSecrets:
- name: docker-registry-secret
---
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
  annotations:
    kubernetes.io/service-account.name: registry-sa
type: kubernetes.io/service-account-token
```

### Pod with ServiceAccount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  serviceAccountName: my-service-account
  automountServiceAccountToken: true
  containers:
  - name: app
    image: nginx:1.21
    env:
    - name: KUBERNETES_TOKEN
      valueFrom:
        secretKeyRef:
          name: my-service-account-token
          key: token
```

---

## 🔹 Roles & ClusterRoles

### Role (Namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
  labels:
    app: monitoring
    component: rbac
rules:
# Basic resource permissions
- apiGroups: [""]           # Core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Specific resource names
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["web-pod-1", "web-pod-2"]
  verbs: ["delete"]

# Subresources
- apiGroups: [""]
  resources: ["pods/log", "pods/status"]
  verbs: ["get"]

# Multiple API groups
- apiGroups: ["apps", "extensions"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch"]
```

### ClusterRole (Cluster-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-custom
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
# Cluster-wide resources
- apiGroups: [""]
  resources: ["nodes", "persistentvolumes"]
  verbs: ["get", "list", "watch"]

# All resources in API group
- apiGroups: ["apps"]
  resources: ["*"]
  verbs: ["*"]

# Non-resource URLs
- nonResourceURLs: ["/healthz", "/version", "/metrics"]
  verbs: ["get"]

# Custom resources
- apiGroups: ["custom.io"]
  resources: ["customresources"]
  verbs: ["get", "list", "create", "update", "delete"]
```

### Common Verbs and Resources

```yaml
# Standard Verbs:
- get          # Retrieve single resource
- list         # Retrieve multiple resources
- watch        # Watch for changes
- create       # Create new resource
- update       # Update existing resource
- patch        # Partially update resource
- delete       # Delete resource
- deletecollection  # Delete multiple resources

# Core Resources (apiGroups: [""]):
- pods, services, endpoints
- configmaps, secrets
- persistentvolumes, persistentvolumeclaims
- nodes, namespaces
- events, serviceaccounts

# Apps Resources (apiGroups: ["apps"]):
- deployments, replicasets, daemonsets
- statefulsets

# RBAC Resources (apiGroups: ["rbac.authorization.k8s.io"]):
- roles, clusterroles
- rolebindings, clusterrolebindings
```

---

## 🔹 RoleBindings & ClusterRoleBindings

### RoleBinding (Namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
  labels:
    app: monitoring
subjects:
# Bind to user
- kind: User
  name: "alice@company.com"
  apiGroup: rbac.authorization.k8s.io

# Bind to group
- kind: Group
  name: "developers"
  apiGroup: rbac.authorization.k8s.io

# Bind to service account
- kind: ServiceAccount
  name: monitoring-sa
  namespace: production

# Reference to Role
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding (Cluster-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
  labels:
    rbac.authorization.k8s.io/autoupdate: "true"
subjects:
- kind: User
  name: "admin@company.com"
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: "cluster-admins"
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Cross-Namespace Binding

```yaml
# ClusterRole bound to namespace-specific subjects
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: admin-sa
  namespace: kube-system  # Different namespace
roleRef:
  kind: ClusterRole      # ClusterRole in RoleBinding
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔹 Practical Examples

### 1. Developer Role (Namespace Access)

```yaml
# ServiceAccount for developers
apiVersion: v1
kind: ServiceAccount
metadata:
  name: developer-sa
  namespace: development
---
# Role with development permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log", "pods/exec"]
  verbs: ["get", "create"]
---
# Bind role to developers
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: Group
  name: "developers"
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: developer-sa
  namespace: development
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### 2. Monitoring System (Cluster-wide Read)

```yaml
# ServiceAccount for monitoring
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
  namespace: monitoring
---
# ClusterRole for monitoring
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
- nonResourceURLs: ["/metrics", "/healthz"]
  verbs: ["get"]
---
# Bind cluster role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-binding
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3. CI/CD Pipeline (Deployment Access)

```yaml
# ServiceAccount for CI/CD
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-sa
  namespace: cicd
---
# ClusterRole for deployments
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-deployer
rules:
# Core resources
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
# Apps resources
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
# Ingress resources
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
# Read-only cluster info
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
---
# Bind to specific namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-production-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: cicd-sa
  namespace: cicd
roleRef:
  kind: ClusterRole
  name: cicd-deployer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-staging-binding
  namespace: staging
subjects:
- kind: ServiceAccount
  name: cicd-sa
  namespace: cicd
roleRef:
  kind: ClusterRole
  name: cicd-deployer
  apiGroup: rbac.authorization.k8s.io
```

### 4. Security Scanner (Read-only Access)

```yaml
# ServiceAccount for security scanning
apiVersion: v1
kind: ServiceAccount
metadata:
  name: security-scanner-sa
  namespace: security
---
# ClusterRole for security scanning
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: security-scanner
rules:
# Read all resources for security analysis
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list"]
# Read cluster-level resources
- apiGroups: [""]
  resources: ["nodes", "persistentvolumes"]
  verbs: ["get", "list"]
# Read RBAC configuration
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "clusterroles", "rolebindings", "clusterrolebindings"]
  verbs: ["get", "list"]
# Read network policies
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["get", "list"]
---
# Bind cluster role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: security-scanner-binding
subjects:
- kind: ServiceAccount
  name: security-scanner-sa
  namespace: security
roleRef:
  kind: ClusterRole
  name: security-scanner
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔹 Best Practices

### 1. Principle of Least Privilege

```yaml
# Bad: Too broad permissions
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Good: Specific permissions
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update"]
```

### 2. Use Built-in ClusterRoles

```yaml
# Leverage existing roles
roleRef:
  kind: ClusterRole
  name: view          # Read-only access
  # name: edit        # Read-write access (no RBAC)
  # name: admin       # Full access to namespace
  # name: cluster-admin  # Full cluster access
```

### 3. Namespace Isolation

```yaml
# Separate environments with namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: development  # Only development namespace
subjects:
- kind: Group
  name: "dev-team"
roleRef:
  kind: ClusterRole
  name: edit
```

### 4. ServiceAccount Management

```yaml
# Disable auto-mounting when not needed
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
automountServiceAccountToken: false

# Or per-pod basis
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false
```

### 5. Regular Auditing

```bash
# Check permissions
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa

# List all bindings
kubectl get rolebindings,clusterrolebindings -A

# Describe specific binding
kubectl describe rolebinding my-binding -n my-namespace
```

---

## 🔹 Troubleshooting

### Common RBAC Issues

#### 1. Permission Denied Errors

```bash
# Check current permissions
kubectl auth can-i create pods
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa

# Check what you can do
kubectl auth can-i --list

# Check specific resource
kubectl auth can-i get pods --subresource=log
```

#### 2. ServiceAccount Token Issues

```bash
# Check if token is mounted
kubectl exec -it my-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/

# Verify token
kubectl exec -it my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Test API access
kubectl exec -it my-pod -- curl -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc/api/v1/namespaces/default/pods
```

#### 3. Binding Issues

```bash
# Check role bindings
kubectl get rolebindings -A | grep my-user

# Describe binding
kubectl describe rolebinding my-binding

# Check subjects
kubectl get rolebinding my-binding -o yaml | grep -A 10 subjects
```

### Debugging Commands

```bash
# List all RBAC resources
kubectl get roles,clusterroles,rolebindings,clusterrolebindings -A

# Check specific permissions
kubectl auth can-i create deployments --as=system:serviceaccount:default:my-sa

# Whoami equivalent
kubectl auth whoami

# Check effective permissions
kubectl describe clusterrolebinding cluster-admin

# Audit RBAC configuration
kubectl get clusterrolebindings -o wide
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: Explain the difference between Role and ClusterRole.**
- **Role**: Namespace-scoped permissions
- **ClusterRole**: Cluster-wide permissions, can access cluster resources like nodes

**Q2: What is the difference between RoleBinding and ClusterRoleBinding?**
- **RoleBinding**: Grants permissions within a namespace
- **ClusterRoleBinding**: Grants permissions cluster-wide

**Q3: Can you bind a ClusterRole with a RoleBinding?**
- Yes, ClusterRole bound with RoleBinding grants permissions only within that namespace

### Technical Questions

**Q4: How do you check if a user has specific permissions?**
```bash
kubectl auth can-i create pods --as=user@company.com
kubectl auth can-i get secrets --as=system:serviceaccount:default:my-sa
```

**Q5: What happens when a pod doesn't have a ServiceAccount specified?**
- Uses the `default` ServiceAccount in the same namespace
- Gets minimal permissions (usually none)

**Q6: How do you create a read-only user for a specific namespace?**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-binding
  namespace: production
subjects:
- kind: User
  name: "readonly@company.com"
roleRef:
  kind: ClusterRole
  name: view
```

### Scenario-Based Questions

**Q7: Design RBAC for a multi-tenant cluster.**
```yaml
# Separate namespaces per tenant
# Tenant-specific ServiceAccounts
# RoleBindings scoped to tenant namespaces
# NetworkPolicies for network isolation
# ResourceQuotas for resource limits
```

**Q8: How do you grant a CI/CD system deployment permissions?**
```yaml
# Create ServiceAccount for CI/CD
# Create ClusterRole with deployment permissions
# Use RoleBindings for specific namespaces
# Avoid cluster-admin permissions
```

**Q9: Troubleshoot: Pod can't access Kubernetes API.**
```bash
# Check ServiceAccount exists
kubectl get sa my-sa

# Check token is mounted
kubectl describe pod my-pod | grep -A 5 Mounts

# Check RBAC permissions
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa

# Check API server connectivity
kubectl exec -it my-pod -- nslookup kubernetes.default.svc.cluster.local
```

### Security Questions

**Q10: What are RBAC security best practices?**
- Principle of least privilege
- Regular permission audits
- Use built-in roles when possible
- Namespace isolation
- Disable auto-mounting when not needed
- Monitor RBAC changes
- Use groups instead of individual users
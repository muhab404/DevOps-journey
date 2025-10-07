# Complete ArgoCD Deep Dive Guide

## What is GitOps

GitOps is a deployment methodology where Git repositories serve as the single source of truth for infrastructure and application configurations. Changes are made through Git commits, and automated tools ensure the live environment matches the Git state.

**GitOps Principles:**
- Declarative configuration stored in Git
- Versioned and immutable deployments
- Automated deployment and rollback
- Continuous monitoring and drift detection

## Introduction to Argo CD

ArgoCD is a Kubernetes-native GitOps continuous delivery tool that automatically synchronizes applications with their Git repository configurations.

**Key Benefits:**
- Declarative GitOps workflow
- Application definitions, configurations, and environments are declarative
- Application deployment and lifecycle management are automated
- Auditable and easy to understand

## Core Concepts

**Application:** A group of Kubernetes resources as defined by a manifest
**Project:** Provides a logical grouping of applications
**Repository:** Git repository containing application manifests
**Target State:** Desired state as defined in Git
**Live State:** Actual state in the Kubernetes cluster
**Sync Status:** Whether live state matches target state
**Health Status:** Health of the application resources

## Argo CD Architecture Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Git Repo      │    │   ArgoCD        │    │   Kubernetes    │
│                 │    │                 │    │   Cluster       │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │ App Configs │ │◄───┤ │ Repo Server │ │    │ │ Deployments │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    │ ┌─────────────┐ │    │ ┌─────────────┐ │
                       │ │ API Server  │ │◄───┤ │ Services    │ │
                       │ └─────────────┘ │    │ └─────────────┘ │
                       │ ┌─────────────┐ │    │ ┌─────────────┐ │
                       │ │ Controller  │ │────┤ │ ConfigMaps  │ │
                       │ └─────────────┘ │    │ └─────────────┘ │
                       └─────────────────┘    └─────────────────┘
```

## Installation Options

### Non-HA Setup

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### Getting Initial Admin Password

```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Change admin password (recommended)
argocd account update-password
```

### Accessing ArgoCD Server

```bash
# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access via LoadBalancer (production)
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Access via Ingress
```

**Ingress Example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-secret
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

### ArgoCD CLI Installation

```bash
# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# macOS
brew install argocd

# Login
argocd login argocd.example.com
```

## Defining Applications

### Creating Applications (YAML)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Creating Applications (CLI)

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## Tools Detection

ArgoCD automatically detects:
- **Helm** charts (Chart.yaml present)
- **Kustomize** (kustomization.yaml present)
- **Plain YAML** manifests

## Helm Options

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-app
spec:
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: 13.2.23
    helm:
      releaseName: my-nginx
      parameters:
      - name: service.type
        value: LoadBalancer
      - name: replicaCount
        value: "3"
      valueFiles:
      - values-production.yaml
      values: |
        service:
          type: LoadBalancer
        replicaCount: 3
```

## Directory Options

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: directory-app
spec:
  source:
    repoURL: https://github.com/user/repo
    path: manifests
    targetRevision: HEAD
    directory:
      recurse: true
      include: "*.yaml"
      exclude: "secret-*.yaml"
```

## Kustomize Options

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-app
spec:
  source:
    repoURL: https://github.com/user/repo
    path: overlays/production
    targetRevision: HEAD
    kustomize:
      namePrefix: prod-
      nameSuffix: -v1
      images:
      - nginx:1.21.0
      commonLabels:
        environment: production
      patches:
      - target:
          kind: Deployment
          name: nginx
        patch: |-
          - op: replace
            path: /spec/replicas
            value: 3
```

## Multiple Sources for an Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-source-app
spec:
  sources:
  - repoURL: https://github.com/user/helm-charts
    chart: my-app
    targetRevision: 1.0.0
    helm:
      valueFiles:
      - $values/values.yaml
  - repoURL: https://github.com/user/config-repo
    targetRevision: HEAD
    ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

## Projects

### Basic Project

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production applications
  sourceRepos:
  - 'https://github.com/company/*'
  - 'https://charts.bitnami.com/bitnami'
  destinations:
  - namespace: 'prod-*'
    server: https://kubernetes.default.svc
  - namespace: production
    server: https://prod-cluster.example.com
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRole
  namespaceResourceWhitelist:
  - group: ''
    kind: ConfigMap
  - group: 'apps'
    kind: Deployment
```

### Project with Roles

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
spec:
  roles:
  - name: admin
    description: Admin role for production
    policies:
    - p, proj:production:admin, applications, *, production/*, allow
    - p, proj:production:admin, repositories, *, *, allow
    groups:
    - company:production-admins
  - name: developer
    description: Developer role for production
    policies:
    - p, proj:production:developer, applications, get, production/*, allow
    - p, proj:production:developer, applications, sync, production/*, allow
    groups:
    - company:developers
```

## Private Git Repositories

### HTTPS with Token

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/private/repo
  password: ghp_xxxxxxxxxxxxxxxxxxxx
  username: not-used
```

### SSH Key

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: git@github.com:private/repo.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

## Private Helm Repositories

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: helm
  name: private-charts
  url: https://charts.company.com
  username: admin
  password: secret123
```

## Credential Templates

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
type: Opaque
stringData:
  type: git
  url: https://github.com/company
  password: ghp_xxxxxxxxxxxxxxxxxxxx
  username: not-used
```

## Sync Policies

### Automated Sync

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auto-sync-app
spec:
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Revert manual changes
      allowEmpty: false
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Sync Options

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sync-options-app
spec:
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    - RespectIgnoreDifferences=true
    - ApplyOutOfSyncOnly=true
    - Replace=true
```

## Tracking Strategies

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tracking-app
spec:
  source:
    repoURL: https://github.com/user/repo
    # Track specific tag
    targetRevision: v1.2.3
    # Track specific commit
    # targetRevision: abc123def456
    # Track branch HEAD
    # targetRevision: HEAD
    # Track Helm chart version
    # targetRevision: 1.0.0
```

## Diffing Customization

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: diff-app
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
  - group: ""
    kind: Secret
    name: mysecret
    jsonPointers:
    - /data
```

## Sync Phases and Hooks

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-sync-job
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
      - name: pre-sync
        image: alpine:latest
        command: ["/bin/sh", "-c", "echo 'Pre-sync hook executed'"]
      restartPolicy: Never
```

**Available Hooks:**
- PreSync: Before sync operation
- Sync: During sync operation
- PostSync: After sync operation
- SyncFail: When sync fails

## Sync Waves

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-wave-1
  annotations:
    argocd.argoproj.io/sync-wave: "1"
data:
  key: value
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-wave-2
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx:1.21
```

## Defining Kubernetes Clusters

### Remote Clusters

```bash
# Add cluster using service account
argocd cluster add my-cluster-context

# Add cluster with specific name
argocd cluster add my-cluster-context --name production-cluster

# List clusters
argocd cluster list
```

**Cluster Secret Example:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: production-cluster
  server: https://prod-cluster.example.com
  config: |
    {
      "bearerToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9...",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "LS0tLS1CRUdJTi..."
      }
    }
```

## ApplicationSet

### List Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: list-generator
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: engineering-dev
        url: https://dev-cluster.example.com
        environment: dev
      - cluster: engineering-prod
        url: https://prod-cluster.example.com
        environment: prod
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
        kustomize:
          namePrefix: '{{environment}}-'
      destination:
        server: '{{url}}'
        namespace: guestbook
```

### Cluster Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-generator
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  template:
    metadata:
      name: '{{name}}-monitoring'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/monitoring
        targetRevision: HEAD
        path: manifests
      destination:
        server: '{{server}}'
        namespace: monitoring
```

### Git Directory Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-directory-generator
spec:
  generators:
  - git:
      repoURL: https://github.com/company/apps
      revision: HEAD
      directories:
      - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/apps
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

### Matrix Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: matrix-generator
spec:
  generators:
  - matrix:
      generators:
      - git:
          repoURL: https://github.com/company/apps
          revision: HEAD
          directories:
          - path: apps/*
      - clusters:
          selector:
            matchLabels:
              environment: production
  template:
    metadata:
      name: '{{path.basename}}-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/apps
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{path.basename}}'
```

## CI/CD Flow

### Basic CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
        docker tag myapp:${{ github.sha }} myapp:latest
    
    - name: Push to registry
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push myapp:${{ github.sha }}
        docker push myapp:latest
    
    - name: Update manifest
      run: |
        git config --global user.email "ci@company.com"
        git config --global user.name "CI Bot"
        sed -i 's|image: myapp:.*|image: myapp:${{ github.sha }}|' k8s/deployment.yaml
        git add k8s/deployment.yaml
        git commit -m "Update image to ${{ github.sha }}"
        git push
```

## Structuring Git Repositories

### Monorepo Structure
```
├── apps/
│   ├── frontend/
│   │   ├── k8s/
│   │   └── src/
│   └── backend/
│       ├── k8s/
│       └── src/
├── infrastructure/
│   ├── base/
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── prod/
└── argocd/
    ├── applications/
    └── projects/
```

### Multi-repo Structure
```
app-repo/
├── src/
├── Dockerfile
└── .github/workflows/

config-repo/
├── apps/
│   └── myapp/
│       ├── base/
│       └── overlays/
│           ├── dev/
│           ├── staging/
│           └── prod/
└── infrastructure/
```

## App of Apps Pattern

### Root Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/argocd-apps
    targetRevision: HEAD
    path: applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Child Applications Directory Structure

```
applications/
├── infrastructure/
│   ├── ingress-nginx.yaml
│   ├── cert-manager.yaml
│   └── monitoring.yaml
├── platform/
│   ├── database.yaml
│   └── redis.yaml
└── apps/
    ├── frontend.yaml
    ├── backend.yaml
    └── api.yaml
```

### Child Application Example

```yaml
# applications/apps/frontend.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/frontend-config
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

## Best Practices

### Security
- Use RBAC projects to limit access
- Enable SSO integration
- Use private repositories
- Implement resource quotas
- Regular security updates
- Use sealed secrets or external secret management

### Repository Structure
- Separate application code from configuration
- Use environment-specific branches or directories
- Implement proper branching strategy
- Use semantic versioning for releases

### Application Management
- Use meaningful application names
- Implement proper labeling and annotations
- Use sync waves for ordered deployments
- Implement health checks for custom resources

### Monitoring and Observability
- Monitor sync status and health
- Set up alerts for failed syncs
- Use Prometheus metrics
- Implement proper logging

### Performance
- Use resource limits and requests
- Configure Redis for caching
- Use multiple repo servers for high load
- Implement proper cluster sizing

This comprehensive guide covers all the topics you requested with practical examples and YAML configurations. Each section builds upon the previous ones to provide a complete understanding of ArgoCD implementation and best practices.
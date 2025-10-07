# ☁️ GKE & GCP Integration - DevOps Interview Guide

## Table of Contents
1. [GKE Overview](#gke-overview)
2. [GCP Service Integration](#gcp-service-integration)
3. [Workload Identity](#workload-identity)
4. [Storage Integration](#storage-integration)
5. [Networking & Load Balancing](#networking--load-balancing)
6. [Monitoring & Logging](#monitoring--logging)
7. [ArgoCD Integration](#argocd-integration)
8. [GitHub Actions CI/CD](#github-actions-cicd)
9. [Security & IAM](#security--iam)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)

---

## 🔹 GKE Overview

### GKE Cluster Types

```yaml
# GKE Cluster Configuration Options:
Standard Cluster:
  - Full control over nodes
  - Custom node pools
  - Advanced networking
  - Manual scaling

Autopilot Cluster:
  - Fully managed nodes
  - Optimized resource allocation
  - Built-in security
  - Pay-per-pod pricing

Private Cluster:
  - Private node IPs
  - Authorized networks
  - Private Google Access
  - Enhanced security
```

### GKE Cluster Creation

```bash
# Standard GKE cluster with advanced features
gcloud container clusters create production-cluster \
    --zone=us-central1-a \
    --machine-type=e2-standard-4 \
    --num-nodes=3 \
    --enable-autoscaling \
    --min-nodes=1 \
    --max-nodes=10 \
    --enable-autorepair \
    --enable-autoupgrade \
    --enable-network-policy \
    --enable-ip-alias \
    --enable-workload-identity \
    --workload-pool=PROJECT_ID.svc.id.goog \
    --enable-shielded-nodes \
    --disk-type=pd-ssd \
    --disk-size=100GB \
    --preemptible \
    --labels=environment=production,team=devops

# Autopilot cluster
gcloud container clusters create-auto autopilot-cluster \
    --region=us-central1 \
    --enable-network-policy \
    --enable-private-nodes \
    --master-ipv4-cidr-block=172.16.0.0/28
```

### Node Pool Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gke-nodepool-config
data:
  create-nodepool.sh: |
    #!/bin/bash
    # Create specialized node pool
    gcloud container node-pools create compute-pool \
        --cluster=production-cluster \
        --zone=us-central1-a \
        --machine-type=c2-standard-8 \
        --num-nodes=2 \
        --enable-autoscaling \
        --min-nodes=0 \
        --max-nodes=5 \
        --node-taints=workload-type=compute:NoSchedule \
        --node-labels=workload-type=compute \
        --disk-type=pd-ssd \
        --disk-size=200GB \
        --spot
```

---

## 🔹 GCP Service Integration

### Cloud SQL Integration

```yaml
# Cloud SQL Proxy Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-with-cloudsql
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
      serviceAccountName: cloudsql-sa
      containers:
      # Main application container
      - name: web-app
        image: gcr.io/PROJECT_ID/web-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "127.0.0.1"
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: "production"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: cloudsql-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: cloudsql-secret
              key: password
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
      
      # Cloud SQL Proxy sidecar
      - name: cloudsql-proxy
        image: gcr.io/cloudsql-docker/gce-proxy:1.33.2
        command:
        - "/cloud_sql_proxy"
        - "-instances=PROJECT_ID:us-central1:production-db=tcp:5432"
        - "-credential_file=/secrets/service_account.json"
        securityContext:
          runAsNonRoot: true
        volumeMounts:
        - name: cloudsql-instance-credentials
          mountPath: /secrets/
          readOnly: true
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
      
      volumes:
      - name: cloudsql-instance-credentials
        secret:
          secretName: cloudsql-instance-credentials
```

### Cloud Storage Integration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-processor
spec:
  replicas: 2
  selector:
    matchLabels:
      app: file-processor
  template:
    metadata:
      labels:
        app: file-processor
    spec:
      serviceAccountName: gcs-sa
      containers:
      - name: processor
        image: gcr.io/PROJECT_ID/file-processor:latest
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /var/secrets/google/key.json
        - name: GCS_BUCKET
          value: "my-app-storage-bucket"
        - name: GCS_PREFIX
          value: "uploads/"
        volumeMounts:
        - name: google-cloud-key
          mountPath: /var/secrets/google
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
      
      volumes:
      - name: google-cloud-key
        secret:
          secretName: gcs-service-account-key
---
# Service Account for GCS access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gcs-sa
  annotations:
    iam.gke.io/gcp-service-account: gcs-access@PROJECT_ID.iam.gserviceaccount.com
```

### Pub/Sub Integration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pubsub-consumer
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pubsub-consumer
  template:
    metadata:
      labels:
        app: pubsub-consumer
    spec:
      serviceAccountName: pubsub-sa
      containers:
      - name: consumer
        image: gcr.io/PROJECT_ID/pubsub-consumer:latest
        env:
        - name: GOOGLE_CLOUD_PROJECT
          value: "PROJECT_ID"
        - name: PUBSUB_SUBSCRIPTION
          value: "message-processing-sub"
        - name: PUBSUB_TOPIC
          value: "message-processing"
        - name: MAX_MESSAGES
          value: "100"
        - name: ACK_DEADLINE
          value: "60"
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# HPA for Pub/Sub workload
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: pubsub-consumer-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: pubsub-consumer
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: pubsub.googleapis.com|subscription|num_undelivered_messages
        selector:
          matchLabels:
            resource.labels.subscription_id: message-processing-sub
      target:
        type: AverageValue
        averageValue: "10"
```

---

## 🔹 Workload Identity

### Workload Identity Setup

```bash
# Enable Workload Identity on cluster
gcloud container clusters update production-cluster \
    --workload-pool=PROJECT_ID.svc.id.goog \
    --zone=us-central1-a

# Create Google Service Account
gcloud iam service-accounts create gke-workload-sa \
    --display-name="GKE Workload Service Account"

# Grant necessary permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:gke-workload-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# Create Kubernetes Service Account
kubectl create serviceaccount k8s-workload-sa

# Bind Kubernetes SA to Google SA
gcloud iam service-accounts add-iam-policy-binding \
    gke-workload-sa@PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[default/k8s-workload-sa]"

# Annotate Kubernetes Service Account
kubectl annotate serviceaccount k8s-workload-sa \
    iam.gke.io/gcp-service-account=gke-workload-sa@PROJECT_ID.iam.gserviceaccount.com
```

### Workload Identity in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-identity-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workload-identity-app
  template:
    metadata:
      labels:
        app: workload-identity-app
    spec:
      # Use Kubernetes Service Account with Workload Identity
      serviceAccountName: k8s-workload-sa
      containers:
      - name: app
        image: gcr.io/PROJECT_ID/gcp-app:latest
        env:
        # No need for GOOGLE_APPLICATION_CREDENTIALS
        - name: GOOGLE_CLOUD_PROJECT
          value: "PROJECT_ID"
        - name: GCS_BUCKET
          value: "my-secure-bucket"
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        # Application will automatically use Workload Identity
        # for GCP API authentication
```

---

## 🔹 Storage Integration

### Persistent Disk Integration

```yaml
# Storage Class for SSD Persistent Disks
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zones: us-central1-a,us-central1-b,us-central1-c
  replication-type: regional-pd
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# PVC using GCE Persistent Disk
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
---
# StatefulSet with Persistent Disk
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: production
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

### Filestore (NFS) Integration

```yaml
# Filestore CSI Driver
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-csi
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: standard
  network: default
allowVolumeExpansion: true
---
# PVC for shared storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: filestore-csi
  resources:
    requests:
      storage: 1Ti
---
# Deployment using shared storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shared-app
  template:
    metadata:
      labels:
        app: shared-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        volumeMounts:
        - name: shared-data
          mountPath: /shared
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: shared-storage
```

---

## 🔹 Networking & Load Balancing

### GCP Load Balancer Integration

```yaml
# Service with GCP Load Balancer
apiVersion: v1
kind: Service
metadata:
  name: web-service
  annotations:
    # Global Load Balancer
    cloud.google.com/load-balancer-type: "External"
    # Regional Load Balancer
    # cloud.google.com/load-balancer-type: "Internal"
    
    # SSL Certificate
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "projects/PROJECT_ID/global/sslCertificates/my-cert"
    
    # Backend configuration
    cloud.google.com/backend-config: '{"default": "web-backend-config"}'
    
    # NEG (Network Endpoint Groups)
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8080
---
# Backend Configuration
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: web-backend-config
spec:
  # Health check configuration
  healthCheck:
    checkIntervalSec: 10
    timeoutSec: 5
    healthyThreshold: 1
    unhealthyThreshold: 3
    type: HTTP
    requestPath: /health
    port: 8080
  
  # Session affinity
  sessionAffinity:
    affinityType: "CLIENT_IP"
    affinityCookieTtlSec: 3600
  
  # Connection draining
  connectionDraining:
    drainingTimeoutSec: 60
  
  # Custom request headers
  customRequestHeaders:
    headers:
    - "X-Client-Region:{client_region}"
    - "X-Client-City:{client_city}"
```

### Ingress with Google Cloud Load Balancer

```yaml
# Managed Certificate
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: web-ssl-cert
spec:
  domains:
  - example.com
  - www.example.com
---
# Ingress with GCP features
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    # Use GCP Load Balancer
    kubernetes.io/ingress.class: "gce"
    
    # Global static IP
    kubernetes.io/ingress.global-static-ip-name: "web-static-ip"
    
    # Managed SSL certificate
    networking.gke.io/managed-certificates: "web-ssl-cert"
    
    # HTTPS redirect
    kubernetes.io/ingress.allow-http: "false"
    
    # Backend configuration
    cloud.google.com/backend-config: '{"default": "web-backend-config"}'
    
    # Frontend configuration
    networking.gke.io/v1beta1.FrontendConfig: "web-frontend-config"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api/*
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: web-service
            port:
              number: 80
---
# Frontend Configuration
apiVersion: networking.gke.io/v1beta1
kind: FrontendConfig
metadata:
  name: web-frontend-config
spec:
  # SSL Policy
  sslPolicy: "modern-ssl-policy"
  
  # Redirect HTTP to HTTPS
  redirectToHttps:
    enabled: true
    responseCodeName: MOVED_PERMANENTLY_DEFAULT
```

---

## 🔹 Monitoring & Logging

### Google Cloud Operations Integration

```yaml
# Deployment with GCP monitoring annotations
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitored-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: monitored-app
  template:
    metadata:
      labels:
        app: monitored-app
      annotations:
        # Enable Prometheus scraping
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: gcr.io/PROJECT_ID/monitored-app:latest
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: GOOGLE_CLOUD_PROJECT
          value: "PROJECT_ID"
        - name: ENABLE_STACKDRIVER
          value: "true"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Custom Metrics for HPA

```yaml
# HPA with custom Stackdriver metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metrics-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: monitored-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Custom Stackdriver metric
  - type: External
    external:
      metric:
        name: custom.googleapis.com|app|requests_per_second
        selector:
          matchLabels:
            resource.labels.project_id: PROJECT_ID
            resource.labels.cluster_name: production-cluster
            metric.labels.app: monitored-app
      target:
        type: AverageValue
        averageValue: "100"
  
  # Pub/Sub queue depth
  - type: External
    external:
      metric:
        name: pubsub.googleapis.com|subscription|num_undelivered_messages
        selector:
          matchLabels:
            resource.labels.subscription_id: app-queue
      target:
        type: AverageValue
        averageValue: "10"
```

---

## 🔹 ArgoCD Integration

### ArgoCD Installation on GKE

```yaml
# ArgoCD namespace
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
---
# ArgoCD with GKE-specific configurations
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-server-config
  namespace: argocd
data:
  url: https://argocd.example.com
  application.instanceLabelKey: argocd.argoproj.io/instance
  server.rbac.log.enforce.enable: "true"
  exec.enabled: "false"
  admin.enabled: "true"
  timeout.reconciliation: 180s
  
  # OIDC configuration for Google OAuth
  oidc.config: |
    name: Google
    issuer: https://accounts.google.com
    clientId: $oidc.google.clientId
    clientSecret: $oidc.google.clientSecret
    requestedScopes: ["openid", "profile", "email"]
    requestedIDTokenClaims: {"groups": {"essential": true}}
  
  # Repository credentials
  repositories: |
    - type: git
      url: https://github.com/company/k8s-manifests
      passwordSecret:
        name: github-secret
        key: password
      usernameSecret:
        name: github-secret
        key: username
    - type: helm
      url: https://charts.bitnami.com/bitnami
      name: bitnami
---
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-manifests
    targetRevision: HEAD
    path: apps/web-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### ArgoCD with Workload Identity

```yaml
# Service Account for ArgoCD with Workload Identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    iam.gke.io/gcp-service-account: argocd-sa@PROJECT_ID.iam.gserviceaccount.com
---
# ArgoCD Application Set for multi-environment
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-env-apps
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  - git:
      repoURL: https://github.com/company/k8s-manifests
      revision: HEAD
      directories:
      - path: apps/*/overlays/*
  template:
    metadata:
      name: '{{path.basename}}-{{path[1]}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/k8s-manifests
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{path[1]}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## 🔹 GitHub Actions CI/CD

### GitHub Actions Workflow for GKE

```yaml
# .github/workflows/deploy-to-gke.yml
name: Deploy to GKE

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  GKE_CLUSTER: production-cluster
  GKE_ZONE: us-central1-a
  DEPLOYMENT_NAME: web-app
  IMAGE: web-app

jobs:
  setup-build-publish-deploy:
    name: Setup, Build, Publish, and Deploy
    runs-on: ubuntu-latest
    environment: production

    permissions:
      contents: read
      id-token: write

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    # Authenticate with Google Cloud using Workload Identity Federation
    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v1
      with:
        workload_identity_provider: projects/${{ secrets.GCP_PROJECT_NUMBER }}/locations/global/workloadIdentityPools/github-pool/providers/github-provider
        service_account: github-actions@${{ secrets.GCP_PROJECT_ID }}.iam.gserviceaccount.com

    # Setup gcloud CLI
    - name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@v1

    # Configure Docker to use gcloud as credential helper
    - name: Configure Docker
      run: |-
        gcloud --quiet auth configure-docker

    # Get GKE credentials
    - name: Get GKE credentials
      run: |-
        gcloud container clusters get-credentials "$GKE_CLUSTER" --zone "$GKE_ZONE"

    # Build Docker image
    - name: Build Docker image
      run: |-
        docker build \
          --tag "gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA" \
          --build-arg GITHUB_SHA="$GITHUB_SHA" \
          --build-arg GITHUB_REF="$GITHUB_REF" \
          .

    # Push Docker image to Google Container Registry
    - name: Publish Docker image
      run: |-
        docker push "gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA"

    # Set up Kustomize
    - name: Set up Kustomize
      run: |-
        curl -sfLo kustomize https://github.com/kubernetes-sigs/kustomize/releases/download/v3.1.0/kustomize_3.1.0_linux_amd64
        chmod u+x ./kustomize

    # Deploy to GKE
    - name: Deploy to GKE
      run: |-
        ./kustomize edit set image gcr.io/PROJECT_ID/IMAGE:TAG=gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA
        ./kustomize build . | kubectl apply -f -
        kubectl rollout status deployment/$DEPLOYMENT_NAME
        kubectl get services -o wide

    # Run tests
    - name: Run integration tests
      run: |-
        kubectl wait --for=condition=available --timeout=300s deployment/$DEPLOYMENT_NAME
        # Add your integration tests here
        curl -f http://$(kubectl get service $DEPLOYMENT_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/health

    # Notify ArgoCD to sync
    - name: Trigger ArgoCD sync
      run: |-
        # Update Git repository with new image tag
        git config --global user.email "github-actions@company.com"
        git config --global user.name "GitHub Actions"
        
        # Update kustomization.yaml with new image
        sed -i "s|gcr.io/$PROJECT_ID/$IMAGE:.*|gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA|g" k8s/overlays/production/kustomization.yaml
        
        git add k8s/overlays/production/kustomization.yaml
        git commit -m "Update image to $GITHUB_SHA"
        git push origin main
```

### Multi-Environment Deployment

```yaml
# .github/workflows/multi-env-deploy.yml
name: Multi-Environment Deploy

on:
  push:
    branches: [main, develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [staging, production]
        include:
        - environment: staging
          cluster: staging-cluster
          zone: us-central1-b
          branch: develop
        - environment: production
          cluster: production-cluster
          zone: us-central1-a
          branch: main

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Deploy to ${{ matrix.environment }}
      if: github.ref == format('refs/heads/{0}', matrix.branch)
      env:
        ENVIRONMENT: ${{ matrix.environment }}
        CLUSTER: ${{ matrix.cluster }}
        ZONE: ${{ matrix.zone }}
      run: |
        # Deployment logic here
        echo "Deploying to $ENVIRONMENT"
        
        # Authenticate and deploy
        gcloud container clusters get-credentials $CLUSTER --zone $ZONE
        
        # Apply environment-specific configurations
        kubectl apply -k k8s/overlays/$ENVIRONMENT
        
        # Wait for rollout
        kubectl rollout status deployment/web-app -n $ENVIRONMENT
```

---

## 🔹 Security & IAM

### GKE Security Best Practices

```yaml
# Pod Security Policy (deprecated, use Pod Security Standards)
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
---
# Network Policy for GKE
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gke-security-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Binary Authorization

```yaml
# Binary Authorization Policy
apiVersion: v1
kind: ConfigMap
metadata:
  name: binary-authorization-policy
data:
  policy.yaml: |
    admissionWhitelistPatterns:
    - namePattern: gcr.io/PROJECT_ID/*
    defaultAdmissionRule:
      requireAttestationsBy:
      - projects/PROJECT_ID/attestors/prod-attestor
      enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    clusterAdmissionRules:
      us-central1-a.production-cluster:
        requireAttestationsBy:
        - projects/PROJECT_ID/attestors/prod-attestor
        enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
```

---

## 🔹 Best Practices

### Resource Management

```yaml
# Resource quotas and limits
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "3"
---
# Limit ranges
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
  - max:
      cpu: 2000m
      memory: 4Gi
    min:
      cpu: 50m
      memory: 64Mi
    type: Container
```

### Cost Optimization

```yaml
# Spot instances node pool
apiVersion: v1
kind: ConfigMap
metadata:
  name: cost-optimization-config
data:
  spot-nodepool.sh: |
    #!/bin/bash
    # Create spot instance node pool for cost savings
    gcloud container node-pools create spot-pool \
        --cluster=production-cluster \
        --zone=us-central1-a \
        --machine-type=e2-standard-4 \
        --spot \
        --num-nodes=0 \
        --enable-autoscaling \
        --min-nodes=0 \
        --max-nodes=10 \
        --node-taints=cloud.google.com/gke-spot=true:NoSchedule \
        --node-labels=cloud.google.com/gke-spot=true
---
# Deployment with spot instance toleration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  replicas: 3
  selector:
    matchLabels:
      app: batch-processor
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      # Tolerate spot instance taints
      tolerations:
      - key: cloud.google.com/gke-spot
        operator: Equal
        value: "true"
        effect: NoSchedule
      
      # Prefer spot instances
      nodeSelector:
        cloud.google.com/gke-spot: "true"
      
      containers:
      - name: processor
        image: gcr.io/PROJECT_ID/batch-processor:latest
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

---

## 🔹 Interview Questions

### GKE Specific Questions

**Q: What are the differences between GKE Standard and Autopilot?**
A: Key differences:
- **Standard**: Full control over nodes, custom configurations, manual scaling
- **Autopilot**: Fully managed, optimized resource allocation, pay-per-pod pricing
- **Standard**: More flexibility, requires more management
- **Autopilot**: Less flexibility, fully automated operations

**Q: How does Workload Identity work in GKE?**
A: Workload Identity allows pods to authenticate as Google Service Accounts:
- Eliminates need for service account keys
- Maps Kubernetes Service Accounts to Google Service Accounts
- Provides secure, automatic credential rotation
- Follows principle of least privilege

**Q: Explain GKE networking options.**
A: GKE networking features:
- **VPC-native**: Uses alias IP ranges, better integration with GCP
- **Routes-based**: Traditional networking, limited scalability
- **Private clusters**: Nodes have private IPs only
- **Authorized networks**: Control API server access

### Integration Questions

**Q: How do you integrate GKE with Cloud SQL securely?**
A: Secure Cloud SQL integration:
- Use Cloud SQL Proxy sidecar container
- Implement Workload Identity for authentication
- Use private IP for Cloud SQL instances
- Store credentials in Kubernetes Secrets
- Enable SSL/TLS connections

**Q: How do you implement CI/CD with GitHub Actions and GKE?**
A: CI/CD pipeline components:
- Use Workload Identity Federation for authentication
- Build and push images to GCR/Artifact Registry
- Use Kustomize or Helm for deployments
- Implement GitOps with ArgoCD
- Add automated testing and rollback capabilities

**Q: How do you monitor GKE workloads?**
A: Comprehensive monitoring approach:
- Google Cloud Operations (Stackdriver) integration
- Custom metrics for HPA scaling
- Prometheus for detailed metrics
- Structured logging with Cloud Logging
- Alerting policies for critical events

This comprehensive guide covers all essential GKE and GCP integration aspects for senior DevOps engineer interviews, including practical examples, security best practices, and real-world CI/CD implementations.
# 🚀 Google Cloud Run Study Guide
*DevOps-Focused Complete Reference*

## Table of Contents
1. [Cloud Run Fundamentals](#cloud-run-fundamentals)
2. [Core Features](#core-features)
3. [Cloud Run Deployments](#cloud-run-deployments)
4. [Cloud Run for Anthos](#cloud-run-for-anthos)
5. [Networking & Integrations](#networking--integrations)
6. [CI/CD with Cloud Run](#cicd-with-cloud-run)
7. [Monitoring & Troubleshooting](#monitoring--troubleshooting)
8. [DevOps Best Practices](#devops-best-practices)

---

## 🔹 Cloud Run Fundamentals

### Introduction to Cloud Run

**Cloud Run** is a fully managed serverless platform that runs containerized applications. It abstracts away all infrastructure management while providing the flexibility of containers.

**"Container to Production in Seconds"** - Core Promise:
- Deploy any container that listens on a port
- Automatic HTTPS endpoints
- Built-in authentication and authorization
- Pay only for requests (scale to zero)

### What is Cloud Run?

**Key Characteristics:**
- **Serverless for containers** - No server management
- **Zero infrastructure management** - Focus on code, not ops
- **Pay-per-use model** - Only pay when serving requests
- **Automatic scaling** - 0 to 1000+ instances
- **Built on Knative** - Open source, portable

**Container Requirements:**
```dockerfile
# Minimal requirements
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 8080
CMD ["npm", "start"]
```

### Service Comparison Matrix

| Feature | Cloud Run | App Engine | Cloud Functions | GKE |
|---------|-----------|------------|-----------------|-----|
| **Container Support** | ✅ Full | ❌ Limited | ❌ No | ✅ Full |
| **Cold Start** | ~1-2s | ~100ms-2min | ~100ms | N/A |
| **Scale to Zero** | ✅ Yes | ✅ Standard only | ✅ Yes | ❌ No |
| **Custom Runtime** | ✅ Any container | ❌ Limited | ❌ Limited | ✅ Full |
| **Request Timeout** | 60min | 60s/24h | 9min | Unlimited |
| **Concurrency** | 1-1000 | 1-80 | 1 | Unlimited |
| **Pricing Model** | CPU/Memory/Requests | Instance hours | Invocations | VM hours |
| **Management Overhead** | Zero | Zero | Zero | High |

### When to Choose Cloud Run

**Choose Cloud Run for:**
- **Containerized microservices** with HTTP traffic
- **API backends** that need flexible scaling
- **Web applications** with variable traffic
- **Batch processing** jobs (Cloud Run Jobs)
- **Legacy app modernization** (containerize and deploy)

**Avoid Cloud Run for:**
- **Persistent connections** (WebSockets, long polling)
- **Stateful applications** requiring local storage
- **High-frequency, low-latency** workloads
- **Complex networking** requirements

### Knative Foundation

**Benefits of Knative:**
- **Open standard** - No vendor lock-in
- **Portability** - Run on any Kubernetes cluster
- **Community-driven** - Broad ecosystem support
- **Consistent API** - Same experience across clouds

**Knative Components:**
```yaml
# Knative Service (simplified)
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/project/app:latest
        ports:
        - containerPort: 8080
```

### Pricing Model Deep Dive

**Billing Components:**
1. **CPU allocation** - vCPU-seconds during request processing
2. **Memory allocation** - GB-seconds during request processing  
3. **Requests** - Number of requests served
4. **Networking** - Egress traffic

**Cost Calculation Example:**
```python
# Monthly cost calculator
def calculate_cloud_run_cost(requests_per_month, avg_duration_ms, cpu_allocation, memory_gb):
    # CPU cost: $0.00002400 per vCPU-second
    cpu_seconds = (requests_per_month * avg_duration_ms / 1000) * cpu_allocation
    cpu_cost = cpu_seconds * 0.00002400
    
    # Memory cost: $0.00000250 per GB-second  
    memory_seconds = (requests_per_month * avg_duration_ms / 1000) * memory_gb
    memory_cost = memory_seconds * 0.00000250
    
    # Request cost: $0.40 per million requests
    request_cost = (requests_per_month / 1000000) * 0.40
    
    return {
        'cpu_cost': cpu_cost,
        'memory_cost': memory_cost, 
        'request_cost': request_cost,
        'total_cost': cpu_cost + memory_cost + request_cost
    }

# Example: 1M requests/month, 200ms avg, 1 vCPU, 512MB
cost = calculate_cloud_run_cost(1000000, 200, 1, 0.5)
print(f"Monthly cost: ${cost['total_cost']:.2f}")
```

---

## 🔹 Core Features

### Language & Dependency Flexibility

**Any Language Support:**
```dockerfile
# Python example
FROM python:3.11-slim
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]

# Go example  
FROM golang:1.19-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
COPY --from=builder /app/main .
CMD ["./main"]

# Java example
FROM openjdk:17-jre-slim
COPY target/app.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

**Binary & Dependency Support:**
- Custom binaries and libraries
- System packages via Dockerfile
- Multi-stage builds for optimization
- Language-specific optimizations

### Portability Benefits

**Container Portability:**
```bash
# Same container runs everywhere
docker build -t my-app .

# Local development
docker run -p 8080:8080 my-app

# Cloud Run deployment
gcloud run deploy --image gcr.io/project/my-app

# Kubernetes deployment
kubectl apply -f k8s-deployment.yaml

# On-premises deployment
docker-compose up
```

**Migration Scenarios:**
- **From VMs** - Containerize existing applications
- **From App Engine** - More flexibility, same serverless benefits
- **From GKE** - Reduce operational overhead
- **Multi-cloud** - Same container, different clouds

### Developer Experience

**Integrated Toolchain:**
```yaml
# Cloud Build integration
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/app', '.']
- name: 'gcr.io/cloud-builders/docker'  
  args: ['push', 'gcr.io/$PROJECT_ID/app']
- name: 'gcr.io/cloud-builders/gcloud'
  args: ['run', 'deploy', 'app', '--image', 'gcr.io/$PROJECT_ID/app', '--region', 'us-central1']
```

**Development Workflow:**
```bash
# Local development with Cloud Code
gcloud auth configure-docker
skaffold dev --port-forward

# Direct deployment
gcloud run deploy --source .

# With Cloud Build
gcloud builds submit --tag gcr.io/project/app
gcloud run deploy --image gcr.io/project/app
```

### Security Features

**IAM Integration:**
```yaml
# Service-level permissions
bindings:
- members:
  - user:developer@company.com
  role: roles/run.developer
- members:  
  - serviceAccount:ci-cd@project.iam.gserviceaccount.com
  role: roles/run.admin
```

**Service-to-Service Authentication:**
```python
import google.auth.transport.requests
import google.oauth2.id_token

def make_authenticated_request(url):
    req = google.auth.transport.requests.Request()
    id_token = google.oauth2.id_token.fetch_id_token(req, url)
    
    headers = {'Authorization': f'Bearer {id_token}'}
    response = requests.get(url, headers=headers)
    return response.json()
```

**VPC Connector Security:**
```bash
# Create VPC connector for private access
gcloud compute networks vpc-access connectors create my-connector \
    --region=us-central1 \
    --subnet=default \
    --subnet-project=my-project \
    --min-instances=2 \
    --max-instances=10
```

### Scaling Configuration

**Concurrency Settings:**
```bash
# Set maximum concurrent requests per instance
gcloud run deploy my-service \
    --image gcr.io/project/app \
    --concurrency 80 \
    --max-instances 100 \
    --min-instances 1
```

**Scaling Behavior:**
```yaml
# service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
  annotations:
    autoscaling.knative.dev/maxScale: "100"
    autoscaling.knative.dev/minScale: "1"
    run.googleapis.com/cpu-throttling: "false"
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "100"
        run.googleapis.com/execution-environment: gen2
    spec:
      containerConcurrency: 80
      containers:
      - image: gcr.io/project/app
        resources:
          limits:
            cpu: "2"
            memory: "2Gi"
```

---

## 🔹 Cloud Run Deployments

### Deploying Container to Cloud Run

**Console Deployment:**
1. Navigate to Cloud Run in GCP Console
2. Click "Create Service"
3. Select container image or deploy from source
4. Configure service settings (CPU, memory, scaling)
5. Set environment variables and secrets
6. Configure networking and security
7. Deploy and test

**gcloud Deployment:**
```bash
# Basic deployment
gcloud run deploy my-service \
    --image gcr.io/project/my-app:latest \
    --region us-central1 \
    --platform managed

# Advanced deployment
gcloud run deploy my-service \
    --image gcr.io/project/my-app:latest \
    --region us-central1 \
    --platform managed \
    --memory 1Gi \
    --cpu 2 \
    --concurrency 100 \
    --max-instances 50 \
    --min-instances 1 \
    --port 8080 \
    --set-env-vars "DATABASE_URL=postgresql://..." \
    --set-secrets "API_KEY=api-key-secret:latest" \
    --allow-unauthenticated
```

### Cart Service Example

**Microservice Architecture:**
```
Frontend (React) → API Gateway → Cart Service (Cloud Run)
                                      ↓
                              Cloud SQL (PostgreSQL)
```

**Cart Service Implementation:**
```python
# app.py
from flask import Flask, request, jsonify
import os
import psycopg2
from google.cloud import secretmanager

app = Flask(__name__)

# Database connection
def get_db_connection():
    return psycopg2.connect(
        host=os.environ['DB_HOST'],
        database=os.environ['DB_NAME'],
        user=os.environ['DB_USER'],
        password=get_secret('db-password')
    )

def get_secret(secret_name):
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{os.environ['PROJECT_ID']}/secrets/{secret_name}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

@app.route('/cart/<user_id>', methods=['GET'])
def get_cart(user_id):
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("SELECT * FROM cart_items WHERE user_id = %s", (user_id,))
    items = cur.fetchall()
    conn.close()
    return jsonify({'items': items})

@app.route('/cart/<user_id>/add', methods=['POST'])
def add_to_cart(user_id):
    data = request.json
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO cart_items (user_id, product_id, quantity) VALUES (%s, %s, %s)",
        (user_id, data['product_id'], data['quantity'])
    )
    conn.commit()
    conn.close()
    return jsonify({'status': 'added'}), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
```

**Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Create non-root user
RUN useradd --create-home --shell /bin/bash app
USER app

EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

**Deployment:**
```bash
# Build and deploy
gcloud builds submit --tag gcr.io/project/cart-service

gcloud run deploy cart-service \
    --image gcr.io/project/cart-service \
    --region us-central1 \
    --set-env-vars "DB_HOST=10.1.2.3,DB_NAME=ecommerce,DB_USER=cart_user,PROJECT_ID=my-project" \
    --set-secrets "db-password=cart-db-password:latest" \
    --vpc-connector my-connector \
    --memory 512Mi \
    --concurrency 50
```

### Deploying from Source

**Cloud Build + GitHub Integration:**
```yaml
# cloudbuild.yaml
steps:
# Build container
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA', '.']

# Push to registry  
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA']

# Deploy to Cloud Run
- name: 'gcr.io/cloud-builders/gcloud'
  args:
  - 'run'
  - 'deploy'
  - 'my-app'
  - '--image'
  - 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA'
  - '--region'
  - 'us-central1'
  - '--platform'
  - 'managed'

# Run tests
- name: 'gcr.io/cloud-builders/curl'
  args: ['https://my-app-hash-uc.a.run.app/health']
```

**GitHub Actions Integration:**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloud Run
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v0
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ secrets.GCP_PROJECT_ID }}
    
    - name: Configure Docker
      run: gcloud auth configure-docker
    
    - name: Build and Deploy
      run: |
        gcloud builds submit --tag gcr.io/${{ secrets.GCP_PROJECT_ID }}/app:$GITHUB_SHA
        gcloud run deploy app \
          --image gcr.io/${{ secrets.GCP_PROJECT_ID }}/app:$GITHUB_SHA \
          --region us-central1 \
          --platform managed
```

### Cloud Run Jobs

**Batch Processing Jobs:**
```bash
# Create a job
gcloud run jobs create data-processor \
    --image gcr.io/project/processor \
    --region us-central1 \
    --memory 2Gi \
    --cpu 2 \
    --max-retries 3 \
    --parallelism 10 \
    --task-count 100

# Execute job
gcloud run jobs execute data-processor --region us-central1

# Schedule with Cloud Scheduler
gcloud scheduler jobs create http process-daily-data \
    --schedule="0 2 * * *" \
    --uri="https://us-central1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/project/jobs/data-processor:run" \
    --http-method=POST \
    --oauth-service-account-email=scheduler@project.iam.gserviceaccount.com
```

**Job Implementation:**
```python
# processor.py
import os
import sys
from google.cloud import storage, bigquery

def process_batch():
    # Get task index from environment
    task_index = int(os.environ.get('CLOUD_RUN_TASK_INDEX', 0))
    task_count = int(os.environ.get('CLOUD_RUN_TASK_COUNT', 1))
    
    # Process subset of data based on task index
    start_id = task_index * 1000
    end_id = start_id + 1000
    
    print(f"Processing records {start_id} to {end_id}")
    
    # Your processing logic here
    process_records(start_id, end_id)
    
    print(f"Task {task_index} completed successfully")

if __name__ == '__main__':
    process_batch()
```

### Environment Variables & Configuration

**Configuration Management:**
```bash
# Environment variables
gcloud run deploy my-service \
    --set-env-vars "ENV=production,DEBUG=false,LOG_LEVEL=info"

# Secrets from Secret Manager
gcloud run deploy my-service \
    --set-secrets "DATABASE_PASSWORD=db-password:latest,API_KEY=external-api-key:1"

# Configuration from file
gcloud run deploy my-service \
    --env-vars-file env.yaml
```

**env.yaml example:**
```yaml
DATABASE_URL: "postgresql://user@host/db"
REDIS_URL: "redis://cache:6379"
LOG_LEVEL: "info"
FEATURE_FLAGS: "new_ui=true,beta_api=false"
```

**Runtime Configuration:**
```python
import os
from google.cloud import secretmanager

class Config:
    def __init__(self):
        self.database_url = os.environ['DATABASE_URL']
        self.redis_url = os.environ.get('REDIS_URL', 'redis://localhost:6379')
        self.log_level = os.environ.get('LOG_LEVEL', 'info')
        self.api_key = self.get_secret('API_KEY')
    
    def get_secret(self, secret_name):
        if secret_name in os.environ:
            return os.environ[secret_name]
        
        client = secretmanager.SecretManagerServiceClient()
        name = f"projects/{os.environ['PROJECT_ID']}/secrets/{secret_name}/versions/latest"
        response = client.access_secret_version(request={"name": name})
        return response.payload.data.decode("UTF-8")

config = Config()
```

### Traffic Splitting & Revisions

**Blue/Green Deployment:**
```bash
# Deploy new revision without traffic
gcloud run deploy my-service \
    --image gcr.io/project/app:v2 \
    --no-traffic \
    --tag blue

# Test new revision
curl https://blue---my-service-hash-uc.a.run.app

# Switch traffic instantly
gcloud run services update-traffic my-service \
    --to-revisions blue=100

# Rollback if needed
gcloud run services update-traffic my-service \
    --to-revisions LATEST=100
```

**Canary Deployment:**
```bash
# Deploy canary with 10% traffic
gcloud run deploy my-service \
    --image gcr.io/project/app:v2 \
    --tag canary

gcloud run services update-traffic my-service \
    --to-revisions canary=10,LATEST=90

# Monitor metrics, then increase traffic
gcloud run services update-traffic my-service \
    --to-revisions canary=50,LATEST=50

# Full rollout
gcloud run services update-traffic my-service \
    --to-revisions canary=100
```

---

## 🔹 Cloud Run for Anthos

### What is Anthos?

**Anthos Overview:**
- **Multi-cloud platform** for hybrid and multi-cloud environments
- **Kubernetes-based** application management
- **Consistent experience** across on-premises, Google Cloud, and other clouds
- **Service mesh** with Istio for advanced networking

**Anthos Components:**
- **Anthos clusters** - Managed Kubernetes
- **Anthos Service Mesh** - Istio-based service mesh
- **Anthos Config Management** - GitOps for configuration
- **Cloud Run for Anthos** - Serverless on Kubernetes

### Cloud Run on Anthos vs Fully Managed

| Feature | Cloud Run (Fully Managed) | Cloud Run for Anthos |
|---------|---------------------------|---------------------|
| **Infrastructure** | Google-managed | Customer Kubernetes cluster |
| **Scaling** | 0 to 1000+ instances | Based on cluster capacity |
| **Networking** | Limited VPC integration | Full Kubernetes networking |
| **Customization** | Standard configurations | Full Kubernetes flexibility |
| **Cost Model** | Pay-per-use | Cluster + usage costs |
| **Portability** | Google Cloud only | Any Kubernetes cluster |

### Deploying to Anthos Clusters

**Cluster Setup:**
```bash
# Create Anthos cluster
gcloud container clusters create my-anthos-cluster \
    --zone us-central1-a \
    --enable-cloud-run-alpha \
    --addons CloudRun \
    --machine-type n1-standard-4 \
    --num-nodes 3

# Get credentials
gcloud container clusters get-credentials my-anthos-cluster --zone us-central1-a
```

**Deploy to Anthos:**
```bash
# Deploy to Anthos cluster
gcloud run deploy my-service \
    --image gcr.io/project/app \
    --cluster my-anthos-cluster \
    --cluster-location us-central1-a \
    --platform gke

# Or use kubectl directly
kubectl apply -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  template:
    spec:
      containers:
      - image: gcr.io/project/app
        ports:
        - containerPort: 8080
EOF
```

**On-Premises Deployment:**
```yaml
# anthos-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
  namespace: production
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "10"
        autoscaling.knative.dev/minScale: "1"
    spec:
      containers:
      - image: my-registry.com/app:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
```

### Benefits of Kubernetes + Serverless

**Hybrid Architecture:**
```yaml
# Traditional Kubernetes workloads
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:13
---
# Serverless Cloud Run services
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: api-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/project/api
```

**Investment Reuse:**
- **Existing Kubernetes expertise** applies directly
- **Same monitoring and logging** tools (Prometheus, Grafana)
- **Consistent networking** with service mesh
- **Gradual migration** from traditional to serverless

---

## 🔹 Networking & Integrations

### Custom Domains & SSL

**Domain Mapping:**
```bash
# Map custom domain
gcloud run domain-mappings create \
    --service my-service \
    --domain api.mycompany.com \
    --region us-central1

# Verify domain ownership
gcloud domains verify api.mycompany.com

# Check SSL certificate status
gcloud run domain-mappings describe \
    --domain api.mycompany.com \
    --region us-central1
```

**DNS Configuration:**
```bash
# Get the ghs.googlehosted.com record
gcloud run domain-mappings describe \
    --domain api.mycompany.com \
    --region us-central1 \
    --format="value(status.resourceRecords[0].rrdata)"

# Add CNAME record in your DNS provider:
# api.mycompany.com CNAME ghs.googlehosted.com
```

### Cloud SQL Integration

**Secure Connection Setup:**
```bash
# Create Cloud SQL instance
gcloud sql instances create my-database \
    --database-version POSTGRES_13 \
    --region us-central1 \
    --tier db-f1-micro

# Create database and user
gcloud sql databases create ecommerce --instance my-database
gcloud sql users create app-user --instance my-database --password [PASSWORD]

# Deploy Cloud Run with Cloud SQL connection
gcloud run deploy my-service \
    --image gcr.io/project/app \
    --add-cloudsql-instances my-project:us-central1:my-database \
    --set-env-vars "DB_HOST=/cloudsql/my-project:us-central1:my-database" \
    --set-secrets "DB_PASSWORD=db-password:latest"
```

**Application Code:**
```python
import os
import sqlalchemy

# Cloud SQL connection
def create_connection():
    if os.environ.get('GAE_ENV') == 'standard':
        # Production - Unix socket
        db_socket_dir = "/cloudsql"
        instance_connection_name = os.environ["CLOUD_SQL_CONNECTION_NAME"]
        socket_path = f"{db_socket_dir}/{instance_connection_name}"
        
        return sqlalchemy.create_engine(
            sqlalchemy.engine.url.URL.create(
                drivername="postgresql+pg8000",
                username=os.environ["DB_USER"],
                password=os.environ["DB_PASSWORD"],
                database=os.environ["DB_NAME"],
                query={"unix_sock": f"{socket_path}/.s.PGSQL.5432"}
            )
        )
    else:
        # Local development - TCP
        return sqlalchemy.create_engine(
            f"postgresql://{os.environ['DB_USER']}:{os.environ['DB_PASSWORD']}@{os.environ['DB_HOST']}/{os.environ['DB_NAME']}"
        )

engine = create_connection()
```

### VPC Connectors

**Private Network Access:**
```bash
# Create VPC connector
gcloud compute networks vpc-access connectors create my-connector \
    --region us-central1 \
    --subnet default \
    --subnet-project my-project \
    --min-instances 2 \
    --max-instances 10 \
    --machine-type e2-micro

# Deploy service with VPC access
gcloud run deploy my-service \
    --image gcr.io/project/app \
    --vpc-connector my-connector \
    --vpc-egress all-traffic
```

**Use Cases:**
- Access **internal databases** (Cloud SQL private IP)
- Connect to **on-premises** resources via VPN
- Reach **internal APIs** not exposed publicly
- Access **Memorystore** (Redis/Memcached)

### CDN Integration

**Cloud CDN Setup:**
```bash
# Create load balancer backend
gcloud compute backend-services create cloud-run-backend \
    --global

# Add Cloud Run NEG
gcloud compute network-endpoint-groups create cloud-run-neg \
    --region us-central1 \
    --network-endpoint-type serverless \
    --cloud-run-service my-service

gcloud compute backend-services add-backend cloud-run-backend \
    --global \
    --network-endpoint-group cloud-run-neg \
    --network-endpoint-group-region us-central1

# Create URL map and proxy
gcloud compute url-maps create cloud-run-map \
    --default-backend-service cloud-run-backend

gcloud compute target-https-proxies create cloud-run-proxy \
    --url-map cloud-run-map \
    --ssl-certificates my-ssl-cert

# Create forwarding rule
gcloud compute forwarding-rules create cloud-run-rule \
    --global \
    --target-https-proxy cloud-run-proxy \
    --ports 443
```

**CDN Configuration:**
```yaml
# Backend service with CDN
apiVersion: compute/v1
kind: BackendService
metadata:
  name: cloud-run-backend
spec:
  backends:
  - group: projects/PROJECT/regions/us-central1/networkEndpointGroups/cloud-run-neg
  enableCDN: true
  cdnPolicy:
    cacheKeyPolicy:
      includeHost: true
      includeProtocol: true
      includeQueryString: false
    defaultTtl: 3600
    maxTtl: 86400
    negativeCaching: true
```

### API Gateway Integration

**API Gateway Configuration:**
```yaml
# api-config.yaml
swagger: '2.0'
info:
  title: My API
  version: 1.0.0
host: api.mycompany.com
schemes:
  - https
paths:
  /users:
    get:
      operationId: getUsers
      x-google-backend:
        address: https://users-service-hash-uc.a.run.app
        protocol: h2
      security:
      - api_key: []
  /orders:
    post:
      operationId: createOrder
      x-google-backend:
        address: https://orders-service-hash-uc.a.run.app
        protocol: h2
      security:
      - firebase: []

securityDefinitions:
  api_key:
    type: apiKey
    name: key
    in: query
  firebase:
    authorizationUrl: ""
    flow: "implicit"
    type: "oauth2"
    x-google-issuer: "https://securetoken.google.com/PROJECT_ID"
    x-google-jwks_uri: "https://www.googleapis.com/service_accounts/v1/metadata/x509/securetoken@system.gserviceaccount.com"
```

**Deploy API Gateway:**
```bash
# Create API config
gcloud api-gateway api-configs create my-config \
    --api my-api \
    --openapi-spec api-config.yaml

# Create gateway
gcloud api-gateway gateways create my-gateway \
    --api my-api \
    --api-config my-config \
    --location us-central1
```

---

## 🔹 CI/CD with Cloud Run

### Cloud Build Pipelines

**Multi-Stage Pipeline:**
```yaml
# cloudbuild.yaml
steps:
# Run tests
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'test-image', '--target', 'test', '.']
- name: 'test-image'
  args: ['python', '-m', 'pytest', 'tests/']

# Build production image
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA', '.']

# Push to registry
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA']

# Deploy to staging
- name: 'gcr.io/cloud-builders/gcloud'
  args:
  - 'run'
  - 'deploy'
  - 'app-staging'
  - '--image'
  - 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA'
  - '--region'
  - 'us-central1'
  - '--tag'
  - 'staging'
  - '--no-traffic'

# Run integration tests
- name: 'gcr.io/cloud-builders/curl'
  args: ['https://staging---app-staging-hash-uc.a.run.app/health']

# Deploy to production (manual approval required)
- name: 'gcr.io/cloud-builders/gcloud'
  args:
  - 'run'
  - 'deploy'
  - 'app-prod'
  - '--image'
  - 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA'
  - '--region'
  - 'us-central1'
  waitFor: ['-']  # Manual trigger

options:
  logging: CLOUD_LOGGING_ONLY
```

**Trigger Configuration:**
```bash
# Create build trigger
gcloud builds triggers create github \
    --repo-name my-app \
    --repo-owner mycompany \
    --branch-pattern "^main$" \
    --build-config cloudbuild.yaml \
    --description "Deploy to Cloud Run on main branch"
```

### GitHub Actions Integration

**Complete Workflow:**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloud Run
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  PROJECT_ID: my-project
  SERVICE_NAME: my-app
  REGION: us-central1

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Python
      uses: actions/setup-python@v2
      with:
        python-version: 3.11
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install pytest
    
    - name: Run tests
      run: pytest tests/

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v0
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ env.PROJECT_ID }}
    
    - name: Configure Docker
      run: gcloud auth configure-docker
    
    - name: Build image
      run: |
        docker build -t gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA .
        docker push gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA
    
    - name: Deploy to Cloud Run
      run: |
        gcloud run deploy $SERVICE_NAME \
          --image gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA \
          --region $REGION \
          --platform managed \
          --allow-unauthenticated
    
    - name: Integration test
      run: |
        SERVICE_URL=$(gcloud run services describe $SERVICE_NAME --region $REGION --format 'value(status.url)')
        curl -f $SERVICE_URL/health || exit 1
```

### GitLab CI Integration

**GitLab Pipeline:**
```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  PROJECT_ID: my-project
  SERVICE_NAME: my-app
  REGION: us-central1

test:
  stage: test
  image: python:3.11
  script:
    - pip install -r requirements.txt pytest
    - pytest tests/

build:
  stage: build
  image: google/cloud-sdk:alpine
  services:
    - docker:dind
  before_script:
    - echo $GCP_SERVICE_KEY | base64 -d > gcp-key.json
    - gcloud auth activate-service-account --key-file gcp-key.json
    - gcloud config set project $PROJECT_ID
    - gcloud auth configure-docker
  script:
    - docker build -t gcr.io/$PROJECT_ID/$SERVICE_NAME:$CI_COMMIT_SHA .
    - docker push gcr.io/$PROJECT_ID/$SERVICE_NAME:$CI_COMMIT_SHA
  only:
    - main

deploy:
  stage: deploy
  image: google/cloud-sdk:alpine
  before_script:
    - echo $GCP_SERVICE_KEY | base64 -d > gcp-key.json
    - gcloud auth activate-service-account --key-file gcp-key.json
    - gcloud config set project $PROJECT_ID
  script:
    - gcloud run deploy $SERVICE_NAME
        --image gcr.io/$PROJECT_ID/$SERVICE_NAME:$CI_COMMIT_SHA
        --region $REGION
        --platform managed
  only:
    - main
```

### Progressive Delivery

**Canary Deployment Automation:**
```bash
#!/bin/bash
# canary-deploy.sh

SERVICE_NAME="my-app"
REGION="us-central1"
NEW_IMAGE="gcr.io/project/app:$1"

echo "Deploying canary version..."

# Deploy new revision with canary tag
gcloud run deploy $SERVICE_NAME \
    --image $NEW_IMAGE \
    --region $REGION \
    --tag canary \
    --no-traffic

# Start with 10% traffic
gcloud run services update-traffic $SERVICE_NAME \
    --region $REGION \
    --to-revisions canary=10,LATEST=90

echo "Monitoring canary for 5 minutes..."
sleep 300

# Check error rate
ERROR_RATE=$(gcloud logging read "
    resource.type=cloud_run_revision AND
    resource.labels.service_name=$SERVICE_NAME AND
    resource.labels.revision_name=~canary AND
    severity>=ERROR
" --limit=100 --format="value(timestamp)" | wc -l)

if [ $ERROR_RATE -lt 5 ]; then
    echo "Canary healthy, promoting to 50%"
    gcloud run services update-traffic $SERVICE_NAME \
        --region $REGION \
        --to-revisions canary=50,LATEST=50
    
    sleep 300
    
    echo "Promoting to 100%"
    gcloud run services update-traffic $SERVICE_NAME \
        --region $REGION \
        --to-revisions canary=100
else
    echo "Canary unhealthy, rolling back"
    gcloud run services update-traffic $SERVICE_NAME \
        --region $REGION \
        --to-revisions LATEST=100
    exit 1
fi
```

---

## 🔹 Monitoring & Troubleshooting

### Logging & Metrics

**Cloud Logging Integration:**
```python
import logging
from google.cloud import logging as cloud_logging

# Setup structured logging
client = cloud_logging.Client()
client.setup_logging()

# Use structured logging
logging.info("Request processed", extra={
    'user_id': user_id,
    'request_id': request_id,
    'duration_ms': duration,
    'status_code': 200
})

# Custom metrics
from google.cloud import monitoring_v3

def record_custom_metric(value):
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{PROJECT_ID}"
    
    series = monitoring_v3.TimeSeries()
    series.metric.type = "custom.googleapis.com/my_app/requests_processed"
    series.resource.type = "cloud_run_revision"
    
    point = series.points.add()
    point.value.int64_value = value
    point.interval.end_time.seconds = int(time.time())
    
    client.create_time_series(name=project_name, time_series=[series])
```

**Log Queries:**
```bash
# View service logs
gcloud logs read "resource.type=cloud_run_revision AND resource.labels.service_name=my-app" --limit=50

# Filter by severity
gcloud logs read "resource.type=cloud_run_revision AND severity>=ERROR" --limit=20

# Search for specific patterns
gcloud logs read "resource.type=cloud_run_revision AND textPayload:timeout" --limit=10

# Export logs to BigQuery
gcloud logging sinks create my-app-logs \
    bigquery.googleapis.com/projects/my-project/datasets/logs \
    --log-filter="resource.type=cloud_run_revision AND resource.labels.service_name=my-app"
```

### Common Issues & Solutions

**1. Container Startup Timeouts:**
```dockerfile
# Problem: Slow container startup
FROM python:3.11-slim

# Solution: Optimize image layers
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Use multi-stage builds
FROM python:3.11-slim as builder
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /root/.local /root/.local
COPY . .
CMD ["python", "app.py"]
```

**2. Concurrency Issues:**
```python
# Problem: Database connection exhaustion
import threading
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

# Solution: Proper connection pooling
engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=5,  # Adjust based on concurrency
    max_overflow=10,
    pool_pre_ping=True
)

# Thread-safe session handling
from contextlib import contextmanager

@contextmanager
def get_db_session():
    session = Session(engine)
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

**3. Memory Issues:**
```python
# Problem: Memory leaks
import gc
import psutil

def monitor_memory():
    process = psutil.Process()
    memory_mb = process.memory_info().rss / 1024 / 1024
    
    if memory_mb > 400:  # 400MB threshold
        logging.warning(f"High memory usage: {memory_mb}MB")
        gc.collect()  # Force garbage collection
    
    return memory_mb

# Add to request handler
@app.after_request
def after_request(response):
    monitor_memory()
    return response
```

**4. Cold Start Optimization:**
```python
# Problem: Slow cold starts
import time

# Solution: Lazy loading and caching
_db_connection = None
_ml_model = None

def get_db_connection():
    global _db_connection
    if _db_connection is None:
        _db_connection = create_connection()
    return _db_connection

def get_ml_model():
    global _ml_model
    if _ml_model is None:
        start_time = time.time()
        _ml_model = load_model()
        load_time = time.time() - start_time
        logging.info(f"Model loaded in {load_time:.2f}s")
    return _ml_model

# Warm-up endpoint
@app.route('/_ah/warmup')
def warmup():
    get_db_connection()
    get_ml_model()
    return 'OK', 200
```

### Debugging Failed Deployments

**Deployment Troubleshooting:**
```bash
# Check deployment status
gcloud run services describe my-service --region us-central1

# View deployment logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=my-service" --limit=20

# Test container locally
docker run -p 8080:8080 gcr.io/project/app:latest

# Check resource limits
gcloud run services describe my-service \
    --region us-central1 \
    --format="value(spec.template.spec.containers[0].resources)"
```

**Health Check Implementation:**
```python
@app.route('/health')
def health_check():
    checks = {
        'database': check_database(),
        'external_api': check_external_api(),
        'memory': check_memory_usage()
    }
    
    all_healthy = all(checks.values())
    status_code = 200 if all_healthy else 503
    
    return jsonify({
        'status': 'healthy' if all_healthy else 'unhealthy',
        'checks': checks,
        'timestamp': datetime.utcnow().isoformat()
    }), status_code

def check_database():
    try:
        with get_db_session() as session:
            session.execute('SELECT 1')
        return True
    except Exception as e:
        logging.error(f"Database check failed: {e}")
        return False
```

---

## 🔹 DevOps Best Practices

### Security Hardening

**Container Security:**
```dockerfile
# Use minimal base images
FROM gcr.io/distroless/python3

# Create non-root user
RUN useradd --create-home --shell /bin/bash app
USER app

# Set security headers
COPY --chown=app:app . /app
WORKDIR /app

# Remove unnecessary packages
RUN apt-get remove -y gcc build-essential
```

**IAM Best Practices:**
```bash
# Create service-specific service account
gcloud iam service-accounts create my-app-sa \
    --display-name "My App Service Account"

# Grant minimal permissions
gcloud projects add-iam-policy-binding my-project \
    --member "serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
    --role "roles/cloudsql.client"

# Deploy with service account
gcloud run deploy my-service \
    --image gcr.io/project/app \
    --service-account my-app-sa@my-project.iam.gserviceaccount.com
```

### Cost Optimization

**Resource Right-Sizing:**
```bash
# Monitor resource usage
gcloud monitoring metrics list --filter="metric.type:run.googleapis.com"

# Optimize based on metrics
gcloud run deploy my-service \
    --cpu 1 \
    --memory 512Mi \
    --concurrency 100 \
    --max-instances 50
```

**Cost Monitoring:**
```python
# Cost tracking
from google.cloud import billing

def track_costs():
    client = billing.CloudBillingClient()
    
    # Get billing account
    billing_account = "billingAccounts/XXXXXX-YYYYYY-ZZZZZZ"
    
    # Query costs for Cloud Run
    query = {
        'time_range': {
            'start_date': {'year': 2024, 'month': 1, 'day': 1},
            'end_date': {'year': 2024, 'month': 1, 'day': 31}
        },
        'group_by': [
            {'key': 'service'},
            {'key': 'sku'}
        ],
        'filter': 'service.description="Cloud Run"'
    }
    
    # Analyze costs and optimize
```

### Performance Optimization

**Application Performance:**
```python
# Request tracing
from google.cloud import trace_v1

tracer = trace_v1.TraceServiceClient()

@app.before_request
def before_request():
    g.start_time = time.time()

@app.after_request  
def after_request(response):
    duration = time.time() - g.start_time
    
    # Log slow requests
    if duration > 1.0:
        logging.warning(f"Slow request: {request.path} took {duration:.2f}s")
    
    # Add performance headers
    response.headers['X-Response-Time'] = f"{duration:.3f}s"
    return response
```

**Caching Strategy:**
```python
from google.cloud import memcache
import redis

# Use Memorystore Redis
redis_client = redis.Redis(host='10.0.0.1', port=6379)

def cached_function(key, ttl=3600):
    def decorator(func):
        def wrapper(*args, **kwargs):
            cache_key = f"{func.__name__}:{key}"
            
            # Try cache first
            cached_result = redis_client.get(cache_key)
            if cached_result:
                return json.loads(cached_result)
            
            # Execute function
            result = func(*args, **kwargs)
            
            # Cache result
            redis_client.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator

@cached_function("user_profile", ttl=1800)
def get_user_profile(user_id):
    # Expensive database query
    return fetch_user_from_db(user_id)
```

This comprehensive Cloud Run study guide provides you with the knowledge and practical examples needed to master Google Cloud Run from a DevOps perspective. The guide covers everything from basic concepts to advanced deployment strategies, monitoring, and optimization techniques.
# ☁️ GCP Scenario-Based Interview Questions - Senior DevOps Guide

## Table of Contents
1. [Compute Engine Scenarios](#compute-engine-scenarios)
2. [GKE Production Issues](#gke-production-issues)
3. [Networking & Load Balancing](#networking--load-balancing)
4. [Storage & Database Issues](#storage--database-issues)
5. [Security & IAM Challenges](#security--iam-challenges)
6. [Monitoring & Logging](#monitoring--logging)
7. [CI/CD & Deployment](#cicd--deployment)
8. [Cost Optimization](#cost-optimization)
9. [Disaster Recovery](#disaster-recovery)
10. [Multi-Cloud & Hybrid](#multi-cloud--hybrid)

---

## 🔹 Compute Engine Scenarios

### Scenario 1: Auto-scaling Web Application Under Heavy Load

**Problem**: Your e-commerce website hosted on Compute Engine is experiencing 500% traffic increase during Black Friday. Current instances are overwhelmed, and manual scaling isn't fast enough.

**Current Setup Issues**:
```bash
# Current problematic setup
gcloud compute instances list
# Shows: 3 instances, all at 95% CPU, response times > 5 seconds
# No auto-scaling configured
# Single zone deployment (risk of zone failure)
```

**Investigation Steps**:
```bash
# Check current instance utilization
gcloud compute instances list --format="table(name,zone,status,machineType)"

# Check instance metrics
gcloud logging read "resource.type=gce_instance AND severity>=WARNING" --limit=50

# Analyze load balancer metrics
gcloud compute backend-services describe web-backend-service --global

# Check current traffic patterns
gcloud monitoring metrics list --filter="metric.type:compute.googleapis.com/instance/cpu/utilization"
```

**Solution - Managed Instance Group with Auto-scaling**:
```bash
# 1. Create instance template with optimized configuration
gcloud compute instance-templates create web-app-template \
    --machine-type=e2-standard-4 \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=50GB \
    --boot-disk-type=pd-ssd \
    --tags=web-server,http-server \
    --metadata-from-file startup-script=startup.sh \
    --scopes=https://www.googleapis.com/auth/cloud-platform

# 2. Create managed instance group
gcloud compute instance-groups managed create web-app-mig \
    --template=web-app-template \
    --size=3 \
    --zone=us-central1-a

# 3. Configure auto-scaling
gcloud compute instance-groups managed set-autoscaling web-app-mig \
    --zone=us-central1-a \
    --max-num-replicas=20 \
    --min-num-replicas=3 \
    --target-cpu-utilization=0.6 \
    --cool-down-period=300

# 4. Create health check
gcloud compute health-checks create http web-app-health-check \
    --port=80 \
    --request-path=/health \
    --check-interval=30s \
    --timeout=10s \
    --healthy-threshold=2 \
    --unhealthy-threshold=3

# 5. Create load balancer
gcloud compute backend-services create web-backend-service \
    --protocol=HTTP \
    --health-checks=web-app-health-check \
    --global

# 6. Add instance group to backend service
gcloud compute backend-services add-backend web-backend-service \
    --instance-group=web-app-mig \
    --instance-group-zone=us-central1-a \
    --balancing-mode=UTILIZATION \
    --max-utilization=0.8 \
    --global
```

**Startup Script (startup.sh)**:
```bash
#!/bin/bash
# Update system
apt-get update
apt-get install -y nginx

# Configure nginx for high performance
cat > /etc/nginx/nginx.conf << 'EOF'
user www-data;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    
    server {
        listen 80;
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        location / {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://backend;
        }
    }
}
EOF

# Start services
systemctl enable nginx
systemctl start nginx

# Install monitoring agent
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
bash add-google-cloud-ops-agent-repo.sh --also-install
```

**Multi-Zone Deployment for High Availability**:
```bash
# Create regional managed instance group
gcloud compute instance-groups managed create web-app-regional-mig \
    --template=web-app-template \
    --size=6 \
    --region=us-central1

# Configure regional auto-scaling
gcloud compute instance-groups managed set-autoscaling web-app-regional-mig \
    --region=us-central1 \
    --max-num-replicas=50 \
    --min-num-replicas=6 \
    --target-cpu-utilization=0.6 \
    --cool-down-period=180

# Add multiple zones for distribution
gcloud compute instance-groups managed set-target-pools web-app-regional-mig \
    --region=us-central1 \
    --target-pools=web-app-pool
```

**Monitoring and Alerting**:
```bash
# Create alerting policy for high CPU
gcloud alpha monitoring policies create --policy-from-file=cpu-alert-policy.yaml
```

```yaml
# cpu-alert-policy.yaml
displayName: "High CPU Usage Alert"
conditions:
  - displayName: "CPU usage above 80%"
    conditionThreshold:
      filter: 'resource.type="gce_instance"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 0.8
      duration: 300s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_MEAN
          crossSeriesReducer: REDUCE_MEAN
          groupByFields:
            - resource.label.instance_id
notificationChannels:
  - projects/PROJECT_ID/notificationChannels/NOTIFICATION_CHANNEL_ID
alertStrategy:
  autoClose: 86400s
```

---

### Scenario 2: Database Performance Issues with Cloud SQL

**Problem**: Your Cloud SQL PostgreSQL instance is experiencing slow query performance, connection timeouts, and occasional downtime during peak hours.

**Investigation**:
```bash
# Check Cloud SQL instance status
gcloud sql instances describe production-db

# Check recent operations
gcloud sql operations list --instance=production-db --limit=10

# Analyze performance insights
gcloud sql instances describe production-db --format="value(settings.insightsConfig)"

# Check current connections
gcloud sql instances describe production-db --format="value(stats.databaseConnections)"
```

**Root Cause Analysis**:
```bash
# Check slow query logs
gcloud logging read "resource.type=cloudsql_database AND severity>=WARNING" \
    --format="table(timestamp,severity,textPayload)" \
    --limit=50

# Analyze query performance
gcloud sql instances describe production-db \
    --format="value(settings.databaseFlags[].name,settings.databaseFlags[].value)"
```

**Solution - Database Optimization**:
```bash
# 1. Upgrade instance for better performance
gcloud sql instances patch production-db \
    --tier=db-custom-8-32768 \
    --storage-size=500GB \
    --storage-type=SSD

# 2. Configure database flags for performance
gcloud sql instances patch production-db \
    --database-flags=shared_preload_libraries=pg_stat_statements,pg_stat_statements.track=all,log_min_duration_statement=1000,log_statement=all

# 3. Create read replicas for read scaling
gcloud sql instances create production-db-replica \
    --master-instance-name=production-db \
    --tier=db-custom-4-16384 \
    --replica-type=READ \
    --region=us-central1

# 4. Configure connection pooling with PgBouncer
gcloud compute instances create pgbouncer-instance \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --machine-type=e2-medium \
    --metadata-from-file startup-script=pgbouncer-setup.sh
```

**PgBouncer Setup Script**:
```bash
#!/bin/bash
# pgbouncer-setup.sh
apt-get update
apt-get install -y pgbouncer postgresql-client

# Configure PgBouncer
cat > /etc/pgbouncer/pgbouncer.ini << 'EOF'
[databases]
production = host=CLOUD_SQL_IP port=5432 dbname=production

[pgbouncer]
listen_port = 5432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid
admin_users = admin
pool_mode = transaction
server_reset_query = DISCARD ALL
max_client_conn = 1000
default_pool_size = 20
max_db_connections = 100
server_lifetime = 3600
server_idle_timeout = 600
EOF

# Start PgBouncer
systemctl enable pgbouncer
systemctl start pgbouncer
```

**Application-Level Optimization**:
```yaml
# Connection pool configuration for applications
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
    url: jdbc:postgresql://pgbouncer-ip:5432/production
```

---

## 🔹 GKE Production Issues

### Scenario 3: GKE Cluster Node Pool Exhaustion

**Problem**: Your GKE cluster is running out of resources. New pods are stuck in "Pending" state, and cluster autoscaler isn't scaling fast enough during traffic spikes.

**Investigation**:
```bash
# Check cluster status
gcloud container clusters describe production-cluster --zone=us-central1-a

# Check node pool status
gcloud container node-pools list --cluster=production-cluster --zone=us-central1-a

# Check pending pods
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# Check resource allocation
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check cluster autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler
```

**Root Cause**:
- Single node pool with limited machine types
- Conservative autoscaler settings
- Resource requests not properly configured
- No preemptible instances for cost optimization

**Solution - Multi-Node Pool Strategy**:
```bash
# 1. Create high-performance node pool
gcloud container node-pools create high-performance-pool \
    --cluster=production-cluster \
    --zone=us-central1-a \
    --machine-type=c2-standard-8 \
    --num-nodes=0 \
    --enable-autoscaling \
    --min-nodes=0 \
    --max-nodes=10 \
    --node-taints=workload-type=compute:NoSchedule \
    --node-labels=workload-type=compute,performance=high \
    --disk-type=pd-ssd \
    --disk-size=100GB

# 2. Create cost-optimized preemptible pool
gcloud container node-pools create preemptible-pool \
    --cluster=production-cluster \
    --zone=us-central1-a \
    --machine-type=e2-standard-4 \
    --preemptible \
    --num-nodes=0 \
    --enable-autoscaling \
    --min-nodes=0 \
    --max-nodes=20 \
    --node-taints=cloud.google.com/gke-preemptible=true:NoSchedule \
    --node-labels=cost-optimized=true

# 3. Create GPU node pool for ML workloads
gcloud container node-pools create gpu-pool \
    --cluster=production-cluster \
    --zone=us-central1-a \
    --machine-type=n1-standard-4 \
    --accelerator=type=nvidia-tesla-t4,count=1 \
    --num-nodes=0 \
    --enable-autoscaling \
    --min-nodes=0 \
    --max-nodes=5 \
    --node-taints=nvidia.com/gpu=true:NoSchedule

# 4. Update cluster autoscaler settings
kubectl patch configmap cluster-autoscaler-status -n kube-system -p '{"data":{"nodes.max":"100","scale-down-delay-after-add":"2m","scale-down-unneeded-time":"2m"}}'
```

**Workload Scheduling Configuration**:
```yaml
# High-performance workload
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compute-intensive-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: compute-intensive
  template:
    metadata:
      labels:
        app: compute-intensive
    spec:
      # Schedule on high-performance nodes
      nodeSelector:
        workload-type: compute
      tolerations:
      - key: workload-type
        operator: Equal
        value: compute
        effect: NoSchedule
      
      containers:
      - name: app
        image: compute-app:v1.0
        resources:
          requests:
            cpu: 2000m
            memory: 4Gi
          limits:
            cpu: 4000m
            memory: 8Gi
---
# Cost-optimized batch workload
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processing
spec:
  template:
    spec:
      # Schedule on preemptible nodes
      nodeSelector:
        cost-optimized: "true"
      tolerations:
      - key: cloud.google.com/gke-preemptible
        operator: Equal
        value: "true"
        effect: NoSchedule
      
      containers:
      - name: batch-processor
        image: batch-processor:v1.0
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
      restartPolicy: Never
---
# ML workload with GPU
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-inference
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-inference
  template:
    metadata:
      labels:
        app: ml-inference
    spec:
      tolerations:
      - key: nvidia.com/gpu
        operator: Equal
        value: "true"
        effect: NoSchedule
      
      containers:
      - name: ml-app
        image: ml-inference:v1.0
        resources:
          requests:
            nvidia.com/gpu: 1
            cpu: 1000m
            memory: 2Gi
          limits:
            nvidia.com/gpu: 1
            cpu: 2000m
            memory: 4Gi
```

**Cluster Monitoring and Alerting**:
```yaml
# Prometheus rule for node resource alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-resource-alerts
spec:
  groups:
  - name: cluster.rules
    rules:
    - alert: NodePoolCapacityHigh
      expr: (kube_node_status_allocatable{resource="cpu"} - kube_node_status_capacity{resource="cpu"}) / kube_node_status_capacity{resource="cpu"} > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node pool capacity running high"
        description: "Node pool {{ $labels.node }} is {{ $value | humanizePercentage }} full"
    
    - alert: PodsPendingTooLong
      expr: kube_pod_status_phase{phase="Pending"} == 1
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Pods pending for too long"
        description: "Pod {{ $labels.pod }} has been pending for more than 10 minutes"
```

---

## 🔹 Networking & Load Balancing

### Scenario 4: Global Load Balancer SSL Certificate Issues

**Problem**: Your global application is experiencing SSL certificate errors in some regions. Users report "certificate not trusted" errors, and some traffic is being served over HTTP instead of HTTPS.

**Investigation**:
```bash
# Check current SSL certificates
gcloud compute ssl-certificates list

# Check load balancer configuration
gcloud compute target-https-proxies describe web-https-proxy

# Check URL map configuration
gcloud compute url-maps describe web-url-map

# Test SSL from different regions
for region in us-central1 europe-west1 asia-southeast1; do
  echo "Testing from $region"
  gcloud compute instances create ssl-test-$region \
    --zone=${region}-a \
    --machine-type=e2-micro \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud
done
```

**Root Cause Analysis**:
```bash
# Check certificate status and expiration
gcloud compute ssl-certificates describe web-ssl-cert \
    --format="value(certificate,privateKey,creationTimestamp,expireTime)"

# Check managed certificate provisioning
gcloud compute ssl-certificates describe managed-ssl-cert \
    --format="value(managed.status,managed.domainStatus)"
```

**Solution - Managed SSL Certificates with Global Load Balancer**:
```bash
# 1. Create managed SSL certificate
gcloud compute ssl-certificates create managed-ssl-cert \
    --domains=example.com,www.example.com,api.example.com \
    --global

# 2. Create global static IP
gcloud compute addresses create web-global-ip --global

# 3. Create health check
gcloud compute health-checks create http web-health-check \
    --port=80 \
    --request-path=/health \
    --check-interval=30s \
    --timeout=10s \
    --healthy-threshold=2 \
    --unhealthy-threshold=3

# 4. Create backend service with multiple regions
gcloud compute backend-services create web-backend-service \
    --protocol=HTTP \
    --health-checks=web-health-check \
    --global \
    --enable-cdn \
    --cache-mode=CACHE_ALL_STATIC

# Add backends from multiple regions
for region in us-central1 europe-west1 asia-southeast1; do
  gcloud compute backend-services add-backend web-backend-service \
    --instance-group=web-mig-$region \
    --instance-group-region=$region \
    --balancing-mode=UTILIZATION \
    --max-utilization=0.8 \
    --capacity-scaler=1.0 \
    --global
done

# 5. Create URL map with HTTPS redirect
gcloud compute url-maps create web-url-map \
    --default-service=web-backend-service

# 6. Create HTTPS target proxy
gcloud compute target-https-proxies create web-https-proxy \
    --url-map=web-url-map \
    --ssl-certificates=managed-ssl-cert

# 7. Create HTTP target proxy for redirect
gcloud compute url-maps create web-redirect-map \
    --default-url-redirect-response-code=301 \
    --default-url-redirect-https-redirect

gcloud compute target-http-proxies create web-http-proxy \
    --url-map=web-redirect-map

# 8. Create forwarding rules
gcloud compute forwarding-rules create web-https-forwarding-rule \
    --address=web-global-ip \
    --global \
    --target-https-proxy=web-https-proxy \
    --ports=443

gcloud compute forwarding-rules create web-http-forwarding-rule \
    --address=web-global-ip \
    --global \
    --target-http-proxy=web-http-proxy \
    --ports=80
```

**Advanced SSL Configuration**:
```bash
# Create SSL policy for modern security
gcloud compute ssl-policies create modern-ssl-policy \
    --profile=MODERN \
    --min-tls-version=1.2

# Apply SSL policy to HTTPS proxy
gcloud compute target-https-proxies update web-https-proxy \
    --ssl-policy=modern-ssl-policy
```

**CDN Configuration for Global Performance**:
```bash
# Configure Cloud CDN with custom cache settings
gcloud compute backend-services update web-backend-service \
    --enable-cdn \
    --cache-mode=CACHE_ALL_STATIC \
    --default-ttl=3600 \
    --max-ttl=86400 \
    --client-ttl=3600 \
    --global

# Create custom cache key policy
gcloud compute backend-services update web-backend-service \
    --cache-key-include-protocol \
    --cache-key-include-host \
    --cache-key-include-query-string \
    --global
```

**Monitoring SSL Certificate Expiration**:
```yaml
# Cloud Monitoring alert for certificate expiration
apiVersion: monitoring.googleapis.com/v1
kind: AlertPolicy
metadata:
  name: ssl-certificate-expiration
spec:
  displayName: "SSL Certificate Expiring Soon"
  conditions:
  - displayName: "Certificate expires in 30 days"
    conditionThreshold:
      filter: 'resource.type="ssl_certificate"'
      comparison: COMPARISON_LESS_THAN
      thresholdValue: 2592000  # 30 days in seconds
      duration: 0s
  notificationChannels:
  - projects/PROJECT_ID/notificationChannels/EMAIL_CHANNEL
  alertStrategy:
    autoClose: 86400s
```

---

## 🔹 Storage & Database Issues

### Scenario 5: Cloud Storage Performance and Cost Issues

**Problem**: Your application is experiencing slow file uploads/downloads, high Cloud Storage costs, and inconsistent performance across different regions.

**Investigation**:
```bash
# Check current storage usage and costs
gsutil du -sh gs://your-bucket-name

# Analyze storage class distribution
gsutil ls -L -b gs://your-bucket-name

# Check access patterns
gcloud logging read "resource.type=gcs_bucket" --limit=100

# Analyze transfer costs
gcloud billing budgets list --billing-account=BILLING_ACCOUNT_ID
```

**Root Cause Analysis**:
```bash
# Check bucket configuration
gsutil ls -L -b gs://your-bucket-name

# Analyze object lifecycle
gsutil lifecycle get gs://your-bucket-name

# Check regional distribution
gsutil ls -L gs://your-bucket-name/** | grep "Location"
```

**Solution - Optimized Storage Strategy**:
```bash
# 1. Create multi-regional bucket for global access
gsutil mb -c STANDARD -l US gs://your-app-global-bucket

# 2. Create regional buckets for specific regions
gsutil mb -c STANDARD -l us-central1 gs://your-app-us-bucket
gsutil mb -c STANDARD -l europe-west1 gs://your-app-eu-bucket
gsutil mb -c STANDARD -l asia-southeast1 gs://your-app-asia-bucket

# 3. Configure lifecycle management
cat > lifecycle.json << 'EOF'
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {"age": 90, "matchesStorageClass": ["NEARLINE"]}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "ARCHIVE"},
        "condition": {"age": 365, "matchesStorageClass": ["COLDLINE"]}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 2555}  # 7 years
      }
    ]
  }
}
EOF

gsutil lifecycle set lifecycle.json gs://your-app-global-bucket

# 4. Enable versioning and configure retention
gsutil versioning set on gs://your-app-global-bucket
gsutil retention set 30d gs://your-app-global-bucket

# 5. Configure CORS for web applications
cat > cors.json << 'EOF'
[
  {
    "origin": ["https://your-domain.com"],
    "method": ["GET", "POST", "PUT", "DELETE"],
    "responseHeader": ["Content-Type", "x-goog-resumable"],
    "maxAgeSeconds": 3600
  }
]
EOF

gsutil cors set cors.json gs://your-app-global-bucket
```

**Application-Level Optimization**:
```python
# Python example for optimized uploads
from google.cloud import storage
import concurrent.futures
import hashlib

class OptimizedStorageClient:
    def __init__(self):
        self.client = storage.Client()
        self.chunk_size = 8 * 1024 * 1024  # 8MB chunks
    
    def upload_with_resumable(self, bucket_name, file_path, blob_name):
        """Upload large files with resumable uploads"""
        bucket = self.client.bucket(bucket_name)
        blob = bucket.blob(blob_name)
        
        # Enable resumable uploads for files > 8MB
        blob.chunk_size = self.chunk_size
        
        with open(file_path, 'rb') as file:
            blob.upload_from_file(file, checksum='md5')
        
        return blob.public_url
    
    def parallel_upload(self, bucket_name, files):
        """Upload multiple files in parallel"""
        with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
            futures = []
            for file_path, blob_name in files:
                future = executor.submit(
                    self.upload_with_resumable, 
                    bucket_name, 
                    file_path, 
                    blob_name
                )
                futures.append(future)
            
            results = []
            for future in concurrent.futures.as_completed(futures):
                results.append(future.result())
            
            return results
```

**CDN Integration for Global Performance**:
```bash
# Create Cloud CDN-enabled load balancer for storage
gcloud compute backend-buckets create storage-backend \
    --gcs-bucket-name=your-app-global-bucket \
    --enable-cdn \
    --cache-mode=CACHE_ALL_STATIC \
    --default-ttl=3600

# Create URL map for storage
gcloud compute url-maps create storage-url-map \
    --default-backend-bucket=storage-backend

# Add path-based routing for different content types
gcloud compute url-maps add-path-matcher storage-url-map \
    --path-matcher-name=storage-matcher \
    --default-backend-bucket=storage-backend \
    --path-rules="/images/*=storage-backend,/videos/*=storage-backend"
```

**Cost Monitoring and Optimization**:
```bash
# Create budget alert for storage costs
gcloud billing budgets create \
    --billing-account=BILLING_ACCOUNT_ID \
    --display-name="Storage Cost Budget" \
    --budget-amount=1000USD \
    --threshold-rules-percent=0.8,0.9,1.0 \
    --threshold-rules-spend-basis=CURRENT_SPEND \
    --all-projects
```

---

## 🔹 Security & IAM Challenges

### Scenario 6: Security Breach - Compromised Service Account

**Problem**: Security team detected unusual API calls from one of your service accounts. Suspicious activities include accessing unauthorized resources and data exfiltration attempts.

**Immediate Response**:
```bash
# 1. Identify the compromised service account
gcloud logging read "protoPayload.authenticationInfo.principalEmail:suspicious-sa@project.iam.gserviceaccount.com" \
    --limit=100 \
    --format="table(timestamp,protoPayload.methodName,protoPayload.resourceName)"

# 2. Disable the service account immediately
gcloud iam service-accounts disable suspicious-sa@project.iam.gserviceaccount.com

# 3. List all keys for the service account
gcloud iam service-accounts keys list \
    --iam-account=suspicious-sa@project.iam.gserviceaccount.com

# 4. Delete all service account keys
for key_id in $(gcloud iam service-accounts keys list \
    --iam-account=suspicious-sa@project.iam.gserviceaccount.com \
    --format="value(name.basename())"); do
  gcloud iam service-accounts keys delete $key_id \
    --iam-account=suspicious-sa@project.iam.gserviceaccount.com \
    --quiet
done

# 5. Check current IAM bindings
gcloud projects get-iam-policy PROJECT_ID \
    --flatten="bindings[].members" \
    --format="table(bindings.role)" \
    --filter="bindings.members:suspicious-sa@project.iam.gserviceaccount.com"
```

**Forensic Analysis**:
```bash
# Analyze all activities from the compromised account
gcloud logging read "protoPayload.authenticationInfo.principalEmail:suspicious-sa@project.iam.gserviceaccount.com" \
    --format="json" > forensic-logs.json

# Check resource access patterns
gcloud logging read "protoPayload.authenticationInfo.principalEmail:suspicious-sa@project.iam.gserviceaccount.com AND protoPayload.methodName:storage.objects.get" \
    --format="table(timestamp,protoPayload.resourceName,protoPayload.request.object)"

# Identify data exfiltration attempts
gcloud logging read "protoPayload.authenticationInfo.principalEmail:suspicious-sa@project.iam.gserviceaccount.com AND protoPayload.methodName:storage.objects.create" \
    --format="table(timestamp,protoPayload.resourceName,protoPayload.response.size)"
```

**Remediation and Hardening**:
```bash
# 1. Create new service account with minimal permissions
gcloud iam service-accounts create secure-app-sa \
    --display-name="Secure Application Service Account" \
    --description="Replacement SA with minimal permissions"

# 2. Apply principle of least privilege
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:secure-app-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# 3. Create custom role with specific permissions
gcloud iam roles create secureAppRole \
    --project=PROJECT_ID \
    --title="Secure App Role" \
    --description="Minimal permissions for secure app" \
    --permissions="storage.objects.get,storage.objects.list,cloudsql.instances.connect"

# 4. Enable audit logging for all services
cat > audit-policy.yaml << 'EOF'
auditConfigs:
- service: allServices
  auditLogConfigs:
  - logType: ADMIN_READ
  - logType: DATA_READ
  - logType: DATA_WRITE
- service: storage.googleapis.com
  auditLogConfigs:
  - logType: ADMIN_READ
  - logType: DATA_READ
  - logType: DATA_WRITE
    exemptedMembers:
    - serviceAccount:secure-app-sa@PROJECT_ID.iam.gserviceaccount.com
EOF

gcloud logging sinks create security-audit-sink \
    bigquery.googleapis.com/projects/PROJECT_ID/datasets/security_audit \
    --log-filter="protoPayload.@type=type.googleapis.com/google.cloud.audit.AuditLog"

# 5. Implement service account key rotation
cat > rotate-keys.sh << 'EOF'
#!/bin/bash
SA_EMAIL="secure-app-sa@PROJECT_ID.iam.gserviceaccount.com"

# Create new key
NEW_KEY=$(gcloud iam service-accounts keys create /tmp/new-key.json \
    --iam-account=$SA_EMAIL \
    --format="value(name.basename())")

# Update application with new key
kubectl create secret generic app-sa-key \
    --from-file=key.json=/tmp/new-key.json \
    --dry-run=client -o yaml | kubectl apply -f -

# Wait for application to pick up new key
sleep 300

# Delete old keys (keep only the new one)
gcloud iam service-accounts keys list --iam-account=$SA_EMAIL \
    --format="value(name.basename())" | \
    grep -v $NEW_KEY | \
    xargs -I {} gcloud iam service-accounts keys delete {} \
    --iam-account=$SA_EMAIL --quiet

rm /tmp/new-key.json
EOF

chmod +x rotate-keys.sh
```

**Enhanced Security Monitoring**:
```yaml
# Cloud Security Command Center integration
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-monitoring
data:
  security-policy.yaml: |
    # Security monitoring rules
    rules:
    - name: "Unusual API Access"
      condition: "protoPayload.methodName:storage.objects.get AND protoPayload.authenticationInfo.principalEmail:*@*.iam.gserviceaccount.com"
      threshold: 1000  # requests per hour
      action: "alert"
    
    - name: "Service Account Key Creation"
      condition: "protoPayload.methodName:google.iam.admin.v1.CreateServiceAccountKey"
      action: "alert"
    
    - name: "IAM Policy Changes"
      condition: "protoPayload.methodName:SetIamPolicy"
      action: "alert"
```

**Workload Identity Implementation**:
```bash
# Enable Workload Identity on GKE cluster
gcloud container clusters update production-cluster \
    --workload-pool=PROJECT_ID.svc.id.goog \
    --zone=us-central1-a

# Create Kubernetes service account
kubectl create serviceaccount secure-app-ksa

# Bind Kubernetes SA to Google SA
gcloud iam service-accounts add-iam-policy-binding \
    secure-app-sa@PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[default/secure-app-ksa]"

# Annotate Kubernetes service account
kubectl annotate serviceaccount secure-app-ksa \
    iam.gke.io/gcp-service-account=secure-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

---

## 🔹 Cost Optimization

### Scenario 7: Unexpected Cloud Bill Spike

**Problem**: Your monthly GCP bill increased by 300% without any apparent reason. Management demands immediate cost reduction and explanation.

**Investigation**:
```bash
# Check current billing data
gcloud billing budgets list --billing-account=BILLING_ACCOUNT_ID

# Analyze cost breakdown by service
gcloud billing accounts list
gcloud alpha billing accounts describe BILLING_ACCOUNT_ID

# Export billing data to BigQuery for analysis
bq mk --dataset PROJECT_ID:billing_analysis
gcloud billing accounts get-iam-policy BILLING_ACCOUNT_ID

# Check resource usage patterns
gcloud compute instances list --format="table(name,zone,machineType,status,creationTimestamp)"
gcloud compute disks list --format="table(name,zone,sizeGb,type,status,creationTimestamp)"
```

**Cost Analysis Queries**:
```sql
-- BigQuery analysis of billing data
-- Top 10 most expensive services
SELECT
  service.description as service_name,
  SUM(cost) as total_cost,
  SUM(usage.amount) as total_usage
FROM `PROJECT_ID.billing_dataset.gcp_billing_export_v1_BILLING_ACCOUNT_ID`
WHERE _PARTITIONTIME >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY service.description
ORDER BY total_cost DESC
LIMIT 10;

-- Cost by project
SELECT
  project.id as project_id,
  SUM(cost) as total_cost
FROM `PROJECT_ID.billing_dataset.gcp_billing_export_v1_BILLING_ACCOUNT_ID`
WHERE _PARTITIONTIME >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY project.id
ORDER BY total_cost DESC;

-- Compute Engine cost analysis
SELECT
  resource.name,
  sku.description,
  SUM(cost) as total_cost,
  SUM(usage.amount) as total_usage
FROM `PROJECT_ID.billing_dataset.gcp_billing_export_v1_BILLING_ACCOUNT_ID`
WHERE service.description = "Compute Engine"
  AND _PARTITIONTIME >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY resource.name, sku.description
ORDER BY total_cost DESC
LIMIT 20;
```

**Immediate Cost Reduction Actions**:
```bash
# 1. Identify and stop unused instances
gcloud compute instances list --filter="status:TERMINATED" \
    --format="value(name,zone)" | \
    while read name zone; do
      echo "Deleting terminated instance: $name in $zone"
      gcloud compute instances delete $name --zone=$zone --quiet
    done

# 2. Find and delete unattached disks
gcloud compute disks list --filter="-users:*" \
    --format="value(name,zone)" | \
    while read name zone; do
      echo "Deleting unattached disk: $name in $zone"
      gcloud compute disks delete $name --zone=$zone --quiet
    done

# 3. Identify oversized instances
gcloud compute instances list \
    --format="table(name,zone,machineType,status)" \
    --filter="machineType:n1-highmem OR machineType:n1-highcpu"

# 4. Convert to preemptible instances where possible
gcloud compute instance-templates create preemptible-template \
    --machine-type=e2-standard-4 \
    --preemptible \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud

# 5. Implement committed use discounts
gcloud compute commitments create cpu-commitment \
    --plan=12-month \
    --region=us-central1 \
    --resources=vcpu=100,memory=400GB
```

**Automated Cost Optimization**:
```bash
# Create cost optimization script
cat > cost-optimizer.sh << 'EOF'
#!/bin/bash

# Stop development instances during off-hours
if [ $(date +%H) -ge 18 ] || [ $(date +%H) -le 8 ]; then
    gcloud compute instances list \
        --filter="labels.environment=development AND status:RUNNING" \
        --format="value(name,zone)" | \
        while read name zone; do
            echo "Stopping dev instance: $name"
            gcloud compute instances stop $name --zone=$zone --async
        done
fi

# Start development instances during business hours
if [ $(date +%H) -eq 8 ]; then
    gcloud compute instances list \
        --filter="labels.environment=development AND status:TERMINATED" \
        --format="value(name,zone)" | \
        while read name zone; do
            echo "Starting dev instance: $name"
            gcloud compute instances start $name --zone=$zone --async
        done
fi

# Clean up old snapshots (keep last 7 days)
gcloud compute snapshots list \
    --filter="creationTimestamp<$(date -d '7 days ago' --iso-8601)" \
    --format="value(name)" | \
    while read snapshot; do
        echo "Deleting old snapshot: $snapshot"
        gcloud compute snapshots delete $snapshot --quiet
    done

# Clean up old images (keep last 10)
gcloud compute images list --no-standard-images \
    --sort-by=~creationTimestamp \
    --format="value(name)" | \
    tail -n +11 | \
    while read image; do
        echo "Deleting old image: $image"
        gcloud compute images delete $image --quiet
    done
EOF

chmod +x cost-optimizer.sh

# Schedule with cron
echo "0 */6 * * * /path/to/cost-optimizer.sh" | crontab -
```

**Right-sizing Recommendations**:
```bash
# Enable Compute Engine recommendations
gcloud recommender recommendations list \
    --project=PROJECT_ID \
    --recommender=google.compute.instance.MachineTypeRecommender \
    --location=us-central1-a

# Apply machine type recommendations
gcloud recommender recommendations list \
    --project=PROJECT_ID \
    --recommender=google.compute.instance.MachineTypeRecommender \
    --location=us-central1-a \
    --format="value(name,content.overview.resourceName,content.overview.currentMachineType,content.overview.recommendedMachineType)"
```

**Budget Alerts and Controls**:
```bash
# Create budget with multiple thresholds
gcloud billing budgets create \
    --billing-account=BILLING_ACCOUNT_ID \
    --display-name="Monthly Budget Alert" \
    --budget-amount=5000USD \
    --threshold-rules-percent=0.5,0.8,0.9,1.0 \
    --threshold-rules-spend-basis=CURRENT_SPEND \
    --all-projects \
    --notification-channels=projects/PROJECT_ID/notificationChannels/CHANNEL_ID

# Create Cloud Function for automated responses
cat > budget-response.py << 'EOF'
import functions_framework
from google.cloud import compute_v1
import json

@functions_framework.http
def budget_alert_handler(request):
    """Handle budget alerts and take automated actions"""
    
    request_json = request.get_json()
    
    if request_json and 'budgetDisplayName' in request_json:
        budget_name = request_json['budgetDisplayName']
        cost_amount = request_json.get('costAmount', 0)
        budget_amount = request_json.get('budgetAmount', 0)
        
        # Calculate percentage spent
        percentage_spent = (cost_amount / budget_amount) * 100
        
        if percentage_spent >= 90:
            # Stop non-production instances
            stop_non_production_instances()
            
        elif percentage_spent >= 80:
            # Send alert to team
            send_alert_to_team(budget_name, percentage_spent)
    
    return 'OK'

def stop_non_production_instances():
    """Stop instances labeled as non-production"""
    compute_client = compute_v1.InstancesClient()
    
    # Implementation to stop non-production instances
    pass

def send_alert_to_team(budget_name, percentage):
    """Send alert to operations team"""
    # Implementation to send alerts
    pass
EOF
```

This comprehensive GCP scenario-based guide covers real-world challenges that senior DevOps engineers face in production environments. Each scenario includes detailed investigation steps, root cause analysis, and complete solutions with cost optimization and security best practices.
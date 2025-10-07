# ☁️ GCP Interview Questions & Solutions - Senior DevOps Guide

## Table of Contents
1. [Compute Engine](#compute-engine)
2. [Google Kubernetes Engine (GKE)](#google-kubernetes-engine-gke)
3. [Networking](#networking)
4. [Storage & Databases](#storage--databases)
5. [Security & IAM](#security--iam)
6. [Monitoring & Logging](#monitoring--logging)
7. [CI/CD & DevOps](#cicd--devops)
8. [Cost Management](#cost-management)
9. [Architecture & Design](#architecture--design)
10. [Troubleshooting](#troubleshooting)

---

## 🔹 Compute Engine

### Q1: How would you design a highly available web application on Compute Engine?

**Answer**: Design a multi-zone, auto-scaling architecture with load balancing.

**Solution**:
```bash
# 1. Create instance template
gcloud compute instance-templates create web-template \
    --machine-type=e2-standard-2 \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=20GB \
    --boot-disk-type=pd-standard \
    --tags=web-server \
    --metadata-from-file startup-script=startup.sh

# 2. Create regional managed instance group
gcloud compute instance-groups managed create web-mig \
    --template=web-template \
    --size=3 \
    --region=us-central1

# 3. Configure autoscaling
gcloud compute instance-groups managed set-autoscaling web-mig \
    --region=us-central1 \
    --max-num-replicas=10 \
    --min-num-replicas=3 \
    --target-cpu-utilization=0.7

# 4. Create health check
gcloud compute health-checks create http web-health-check \
    --port=80 \
    --request-path=/health

# 5. Create load balancer
gcloud compute backend-services create web-backend \
    --protocol=HTTP \
    --health-checks=web-health-check \
    --global

gcloud compute backend-services add-backend web-backend \
    --instance-group=web-mig \
    --instance-group-region=us-central1 \
    --global
```

**Key Points**:
- **Regional MIG**: Distributes instances across multiple zones
- **Auto-scaling**: Handles traffic spikes automatically
- **Health checks**: Ensures only healthy instances receive traffic
- **Load balancer**: Distributes traffic evenly

---

### Q2: Explain the difference between preemptible and spot instances. When would you use each?

**Answer**: 
- **Preemptible**: Legacy, up to 80% cost savings, 24-hour max runtime
- **Spot**: New model, up to 91% savings, no fixed runtime limit

**Use Cases**:
```bash
# Preemptible for batch processing
gcloud compute instances create batch-worker \
    --preemptible \
    --machine-type=e2-standard-4 \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud

# Spot for fault-tolerant workloads
gcloud compute instances create spot-worker \
    --provisioning-model=SPOT \
    --instance-termination-action=STOP \
    --machine-type=e2-standard-4
```

**Best Practices**:
- Use for stateless applications
- Implement graceful shutdown handling
- Combine with persistent storage for data
- Not suitable for critical production workloads

---

### Q3: How do you implement disaster recovery for Compute Engine instances?

**Answer**: Multi-region deployment with automated snapshots and failover.

**Solution**:
```bash
# 1. Create snapshots schedule
gcloud compute resource-policies create snapshot-schedule daily-backup \
    --region=us-central1 \
    --max-retention-days=7 \
    --on-source-disk-delete=keep-auto-snapshots \
    --daily-schedule \
    --start-time=02:00

# 2. Attach policy to disks
gcloud compute disks add-resource-policies production-disk \
    --resource-policies=daily-backup \
    --zone=us-central1-a

# 3. Create cross-region image
gcloud compute images create dr-image \
    --source-disk=production-disk \
    --source-disk-zone=us-central1-a \
    --storage-location=us

# 4. DR instance template
gcloud compute instance-templates create dr-template \
    --machine-type=e2-standard-2 \
    --image=dr-image \
    --region=us-west1

# 5. Automated failover script
cat > failover.sh << 'EOF'
#!/bin/bash
# Check primary region health
if ! gcloud compute instances describe primary-instance --zone=us-central1-a; then
    echo "Primary region down, starting DR instances"
    gcloud compute instance-groups managed resize dr-mig --size=3 --region=us-west1
    gcloud compute forwarding-rules set-target web-forwarding-rule --target-http-proxy=dr-proxy
fi
EOF
```

---

## 🔹 Google Kubernetes Engine (GKE)

### Q4: What are the differences between GKE Standard and Autopilot modes?

**Answer**: 

| Feature | Standard | Autopilot |
|---------|----------|-----------|
| **Node Management** | Manual | Fully managed |
| **Pricing** | Per node | Per pod resource |
| **Security** | Manual hardening | Built-in security |
| **Flexibility** | Full control | Opinionated setup |

**Standard Cluster**:
```bash
gcloud container clusters create standard-cluster \
    --zone=us-central1-a \
    --machine-type=e2-standard-4 \
    --num-nodes=3 \
    --enable-autoscaling \
    --min-nodes=1 \
    --max-nodes=10 \
    --enable-network-policy
```

**Autopilot Cluster**:
```bash
gcloud container clusters create-auto autopilot-cluster \
    --region=us-central1 \
    --enable-network-policy
```

**When to use**:
- **Standard**: Need custom node configurations, specific machine types, or DaemonSets
- **Autopilot**: Want simplified management, built-in security, and cost optimization

---

### Q5: How do you implement Workload Identity in GKE?

**Answer**: Workload Identity allows pods to authenticate as Google Service Accounts without storing keys.

**Implementation**:
```bash
# 1. Enable Workload Identity on cluster
gcloud container clusters update my-cluster \
    --workload-pool=PROJECT_ID.svc.id.goog

# 2. Create Google Service Account
gcloud iam service-accounts create gke-workload-sa \
    --display-name="GKE Workload Service Account"

# 3. Grant necessary permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:gke-workload-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# 4. Create Kubernetes Service Account
kubectl create serviceaccount k8s-sa

# 5. Bind accounts
gcloud iam service-accounts add-iam-policy-binding \
    gke-workload-sa@PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[default/k8s-sa]"

# 6. Annotate K8s Service Account
kubectl annotate serviceaccount k8s-sa \
    iam.gke.io/gcp-service-account=gke-workload-sa@PROJECT_ID.iam.gserviceaccount.com
```

**Pod Configuration**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: workload-identity-test
spec:
  serviceAccountName: k8s-sa
  containers:
  - name: app
    image: google/cloud-sdk:slim
    command: ["sleep", "infinity"]
```

**Benefits**:
- No service account keys to manage
- Automatic credential rotation
- Fine-grained access control
- Enhanced security posture

---

### Q6: How do you troubleshoot a pod that's stuck in "Pending" state?

**Answer**: Systematic approach to identify resource, scheduling, or configuration issues.

**Troubleshooting Steps**:
```bash
# 1. Check pod status and events
kubectl describe pod <pod-name>

# 2. Check node resources
kubectl top nodes
kubectl describe nodes

# 3. Check resource quotas
kubectl describe quota --all-namespaces

# 4. Check persistent volume claims
kubectl get pvc
kubectl describe pvc <pvc-name>

# 5. Check node selectors and taints
kubectl get nodes --show-labels
kubectl describe node <node-name> | grep Taints
```

**Common Causes & Solutions**:

1. **Insufficient Resources**:
```bash
# Check cluster autoscaler
kubectl get events --sort-by=.metadata.creationTimestamp

# Scale node pool
gcloud container node-pools resize default-pool \
    --cluster=my-cluster \
    --zone=us-central1-a \
    --num-nodes=5
```

2. **PVC Issues**:
```bash
# Check storage class
kubectl get storageclass

# Check PV availability
kubectl get pv
```

3. **Node Affinity Issues**:
```yaml
# Fix node selector
spec:
  nodeSelector:
    kubernetes.io/os: linux  # Correct label
```

---

## 🔹 Networking

### Q7: Explain VPC peering vs VPN vs Interconnect. When would you use each?

**Answer**: Different connectivity options for different use cases and requirements.

**VPC Peering**:
```bash
# Create VPC peering
gcloud compute networks peerings create peer-to-vpc2 \
    --network=vpc1 \
    --peer-project=PROJECT_ID \
    --peer-network=vpc2
```
- **Use case**: Connect VPCs within Google Cloud
- **Benefits**: Low latency, high bandwidth, no single point of failure
- **Limitations**: No transitive peering, same project or trusted projects

**Cloud VPN**:
```bash
# Create VPN gateway
gcloud compute vpn-gateways create my-vpn-gateway \
    --network=my-vpc \
    --region=us-central1

# Create VPN tunnel
gcloud compute vpn-tunnels create my-tunnel \
    --peer-address=203.0.113.1 \
    --shared-secret=SECRET \
    --target-vpn-gateway=my-vpn-gateway \
    --region=us-central1
```
- **Use case**: Connect on-premises to Google Cloud
- **Benefits**: Encrypted, cost-effective
- **Limitations**: Internet-dependent, variable bandwidth

**Cloud Interconnect**:
```bash
# Dedicated Interconnect (requires physical connection)
gcloud compute interconnects create my-interconnect \
    --interconnect-type=DEDICATED \
    --link-type=LINK_TYPE_ETHERNET_10G_LR \
    --location=LOCATION

# Partner Interconnect
gcloud compute interconnects create partner-interconnect \
    --interconnect-type=PARTNER \
    --edge-availability-domain=AVAILABILITY_DOMAIN
```
- **Use case**: High bandwidth, low latency to on-premises
- **Benefits**: Dedicated bandwidth, consistent performance
- **Limitations**: Higher cost, setup complexity

---

### Q8: How do you implement a global load balancer with SSL termination?

**Answer**: Use Google Cloud Load Balancer with managed SSL certificates.

**Implementation**:
```bash
# 1. Create global static IP
gcloud compute addresses create web-ip --global

# 2. Create managed SSL certificate
gcloud compute ssl-certificates create web-ssl-cert \
    --domains=example.com,www.example.com \
    --global

# 3. Create health check
gcloud compute health-checks create http web-health-check \
    --port=80 \
    --request-path=/health

# 4. Create backend service
gcloud compute backend-services create web-backend \
    --protocol=HTTP \
    --health-checks=web-health-check \
    --global

# 5. Add instance groups to backend
gcloud compute backend-services add-backend web-backend \
    --instance-group=web-mig-us \
    --instance-group-region=us-central1 \
    --global

gcloud compute backend-services add-backend web-backend \
    --instance-group=web-mig-eu \
    --instance-group-region=europe-west1 \
    --global

# 6. Create URL map
gcloud compute url-maps create web-map \
    --default-service=web-backend

# 7. Create HTTPS proxy
gcloud compute target-https-proxies create web-https-proxy \
    --url-map=web-map \
    --ssl-certificates=web-ssl-cert

# 8. Create forwarding rule
gcloud compute forwarding-rules create web-https-rule \
    --address=web-ip \
    --global \
    --target-https-proxy=web-https-proxy \
    --ports=443

# 9. HTTP to HTTPS redirect
gcloud compute url-maps create web-redirect-map \
    --default-url-redirect-response-code=301 \
    --default-url-redirect-https-redirect

gcloud compute target-http-proxies create web-http-proxy \
    --url-map=web-redirect-map

gcloud compute forwarding-rules create web-http-rule \
    --address=web-ip \
    --global \
    --target-http-proxy=web-http-proxy \
    --ports=80
```

**Advanced Configuration**:
```bash
# Enable Cloud CDN
gcloud compute backend-services update web-backend \
    --enable-cdn \
    --cache-mode=CACHE_ALL_STATIC \
    --default-ttl=3600 \
    --global

# Configure SSL policy
gcloud compute ssl-policies create modern-ssl-policy \
    --profile=MODERN \
    --min-tls-version=1.2

gcloud compute target-https-proxies update web-https-proxy \
    --ssl-policy=modern-ssl-policy
```

---

## 🔹 Storage & Databases

### Q9: Compare Cloud SQL, Cloud Spanner, and Firestore. When would you use each?

**Answer**: Different database solutions for different requirements.

**Cloud SQL**:
```bash
# Create Cloud SQL instance
gcloud sql instances create mysql-instance \
    --database-version=MYSQL_8_0 \
    --tier=db-n1-standard-2 \
    --region=us-central1 \
    --storage-size=100GB \
    --storage-type=SSD \
    --backup-start-time=03:00
```
- **Use case**: Traditional relational databases, existing applications
- **Benefits**: Familiar SQL, ACID compliance, automated backups
- **Limitations**: Regional, limited scaling

**Cloud Spanner**:
```bash
# Create Spanner instance
gcloud spanner instances create spanner-instance \
    --config=regional-us-central1 \
    --description="Production Spanner" \
    --nodes=3

# Create database
gcloud spanner databases create production-db \
    --instance=spanner-instance
```
- **Use case**: Global applications, strong consistency, horizontal scaling
- **Benefits**: Global distribution, ACID at scale, SQL interface
- **Limitations**: Higher cost, complexity

**Firestore**:
```bash
# Enable Firestore
gcloud firestore databases create --region=us-central1
```
- **Use case**: Mobile/web apps, real-time updates, document-based data
- **Benefits**: Real-time sync, offline support, automatic scaling
- **Limitations**: NoSQL limitations, eventual consistency

---

### Q10: How do you implement backup and disaster recovery for Cloud SQL?

**Answer**: Multi-layered approach with automated backups, replicas, and cross-region recovery.

**Implementation**:
```bash
# 1. Configure automated backups
gcloud sql instances patch production-db \
    --backup-start-time=02:00 \
    --backup-location=us \
    --retained-backups-count=30 \
    --retained-transaction-log-days=7

# 2. Create read replica for read scaling
gcloud sql instances create production-db-replica \
    --master-instance-name=production-db \
    --tier=db-n1-standard-2 \
    --replica-type=READ \
    --region=us-central1

# 3. Create cross-region replica for DR
gcloud sql instances create production-db-dr \
    --master-instance-name=production-db \
    --tier=db-n1-standard-2 \
    --replica-type=READ \
    --region=us-west1

# 4. Export data for additional backup
gcloud sql export sql production-db gs://backup-bucket/backup-$(date +%Y%m%d).sql \
    --database=production

# 5. Point-in-time recovery
gcloud sql backups restore BACKUP_ID \
    --restore-instance=production-db-restored \
    --backup-instance=production-db
```

**Disaster Recovery Script**:
```bash
#!/bin/bash
# dr-failover.sh

PRIMARY_INSTANCE="production-db"
DR_INSTANCE="production-db-dr"
REGION="us-west1"

# Check primary instance health
if ! gcloud sql instances describe $PRIMARY_INSTANCE --format="value(state)" | grep -q "RUNNABLE"; then
    echo "Primary instance unhealthy, promoting DR replica"
    
    # Promote read replica to master
    gcloud sql instances promote-replica $DR_INSTANCE
    
    # Update application connection strings
    kubectl patch configmap app-config \
        -p '{"data":{"DB_HOST":"DR_INSTANCE_IP"}}'
    
    # Restart application pods
    kubectl rollout restart deployment/web-app
    
    echo "Failover completed"
fi
```

---

## 🔹 Security & IAM

### Q11: Explain the principle of least privilege in GCP IAM. How do you implement it?

**Answer**: Grant minimum permissions necessary for users/services to perform their tasks.

**Implementation Strategy**:
```bash
# 1. Use predefined roles when possible
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="user:developer@company.com" \
    --role="roles/compute.instanceAdmin.v1"

# 2. Create custom roles for specific needs
gcloud iam roles create customDeveloperRole \
    --project=PROJECT_ID \
    --title="Custom Developer Role" \
    --description="Limited developer access" \
    --permissions="compute.instances.list,compute.instances.get,compute.instances.start,compute.instances.stop"

# 3. Use conditions for fine-grained access
cat > policy.json << 'EOF'
{
  "bindings": [
    {
      "role": "roles/compute.instanceAdmin.v1",
      "members": ["user:developer@company.com"],
      "condition": {
        "title": "Dev Environment Only",
        "description": "Access only to dev environment",
        "expression": "resource.name.startsWith('projects/PROJECT_ID/zones/us-central1-a/instances/dev-')"
      }
    }
  ]
}
EOF

gcloud projects set-iam-policy PROJECT_ID policy.json
```

**Service Account Best Practices**:
```bash
# Create service account with minimal permissions
gcloud iam service-accounts create app-sa \
    --display-name="Application Service Account"

# Grant specific permissions only
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# Use Workload Identity instead of keys
kubectl annotate serviceaccount app-ksa \
    iam.gke.io/gcp-service-account=app-sa@PROJECT_ID.iam.gserviceaccount.com
```

**Regular Audit Process**:
```bash
# List all IAM bindings
gcloud projects get-iam-policy PROJECT_ID \
    --flatten="bindings[].members" \
    --format="table(bindings.role,bindings.members)"

# Check service account usage
gcloud iam service-accounts get-iam-policy app-sa@PROJECT_ID.iam.gserviceaccount.com

# Review permissions
gcloud iam roles describe roles/compute.instanceAdmin.v1
```

---

### Q12: How do you secure secrets in GCP?

**Answer**: Use Secret Manager with proper access controls and rotation.

**Implementation**:
```bash
# 1. Create secret in Secret Manager
echo -n "super-secret-password" | gcloud secrets create db-password \
    --data-file=-

# 2. Grant access to specific service account
gcloud secrets add-iam-policy-binding db-password \
    --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

# 3. Use in Kubernetes with CSI driver
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: SecretProviderClass
metadata:
  name: app-secrets
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/PROJECT_ID/secrets/db-password/versions/latest"
        path: "db-password"
EOF

# 4. Mount in pod
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      serviceAccountName: app-ksa
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
      volumes:
      - name: secrets-store
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "app-secrets"
EOF
```

**Secret Rotation**:
```bash
# Create new version
echo -n "new-secret-password" | gcloud secrets versions add db-password \
    --data-file=-

# Automatic rotation with Cloud Function
cat > rotate-secret.py << 'EOF'
import functions_framework
from google.cloud import secretmanager
import random
import string

@functions_framework.http
def rotate_secret(request):
    client = secretmanager.SecretManagerServiceClient()
    
    # Generate new password
    new_password = ''.join(random.choices(string.ascii_letters + string.digits, k=16))
    
    # Add new version
    parent = "projects/PROJECT_ID/secrets/db-password"
    response = client.add_secret_version(
        request={"parent": parent, "payload": {"data": new_password.encode("UTF-8")}}
    )
    
    return f"New secret version created: {response.name}"
EOF
```

---

## 🔹 Monitoring & Logging

### Q13: How do you set up comprehensive monitoring for a GCP application?

**Answer**: Use Cloud Operations suite with custom metrics, alerts, and dashboards.

**Implementation**:
```bash
# 1. Install monitoring agent on Compute Engine
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
bash add-google-cloud-ops-agent-repo.sh --also-install

# 2. Create custom metric
gcloud logging metrics create error_rate_metric \
    --description="Application error rate" \
    --log-filter='resource.type="gce_instance" AND severity="ERROR"'

# 3. Create alerting policy
cat > alert-policy.yaml << 'EOF'
displayName: "High Error Rate Alert"
conditions:
  - displayName: "Error rate above threshold"
    conditionThreshold:
      filter: 'metric.type="logging.googleapis.com/user/error_rate_metric"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 10
      duration: 300s
notificationChannels:
  - projects/PROJECT_ID/notificationChannels/CHANNEL_ID
EOF

gcloud alpha monitoring policies create --policy-from-file=alert-policy.yaml

# 4. Create dashboard
cat > dashboard.json << 'EOF'
{
  "displayName": "Application Dashboard",
  "mosaicLayout": {
    "tiles": [
      {
        "width": 6,
        "height": 4,
        "widget": {
          "title": "CPU Utilization",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_MEAN"
                    }
                  }
                }
              }
            ]
          }
        }
      }
    ]
  }
}
EOF

gcloud monitoring dashboards create --config-from-file=dashboard.json
```

**Application-Level Monitoring**:
```python
# Python application with custom metrics
from google.cloud import monitoring_v3
import time

def record_custom_metric(value):
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{PROJECT_ID}"
    
    series = monitoring_v3.TimeSeries()
    series.metric.type = "custom.googleapis.com/application/requests"
    series.resource.type = "gce_instance"
    series.resource.labels["instance_id"] = "INSTANCE_ID"
    series.resource.labels["zone"] = "us-central1-a"
    
    now = time.time()
    seconds = int(now)
    nanos = int((now - seconds) * 10 ** 9)
    interval = monitoring_v3.TimeInterval(
        {"end_time": {"seconds": seconds, "nanos": nanos}}
    )
    point = monitoring_v3.Point(
        {"interval": interval, "value": {"double_value": value}}
    )
    series.points = [point]
    
    client.create_time_series(name=project_name, time_series=[series])
```

**Log-based Metrics**:
```bash
# Create log-based metric for response time
gcloud logging metrics create response_time_metric \
    --description="Average response time" \
    --log-filter='resource.type="gce_instance" AND jsonPayload.response_time_ms>0' \
    --value-extractor='EXTRACT(jsonPayload.response_time_ms)'
```

---

### Q14: How do you implement centralized logging for microservices?

**Answer**: Use Cloud Logging with structured logging and proper correlation IDs.

**Implementation**:
```bash
# 1. Configure Fluent Bit for log collection
cat > fluent-bit.conf << 'EOF'
[SERVICE]
    Flush         1
    Log_Level     info
    Daemon        off
    Parsers_File  parsers.conf

[INPUT]
    Name              tail
    Path              /var/log/app/*.log
    Parser            json
    Tag               app.*
    Refresh_Interval  5

[OUTPUT]
    Name        stackdriver
    Match       *
    google_service_credentials /path/to/credentials.json
    resource    gce_instance
EOF

# 2. Deploy Fluent Bit as DaemonSet in GKE
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluent-bit
  template:
    metadata:
      labels:
        name: fluent-bit
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
EOF
```

**Structured Logging Example**:
```python
# Python microservice with structured logging
import logging
import json
import uuid
from flask import Flask, request, g

app = Flask(__name__)

class StructuredLogger:
    def __init__(self):
        self.logger = logging.getLogger(__name__)
        handler = logging.StreamHandler()
        handler.setFormatter(logging.Formatter('%(message)s'))
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)
    
    def log(self, level, message, **kwargs):
        log_entry = {
            'timestamp': time.time(),
            'level': level,
            'message': message,
            'service': 'user-service',
            'version': '1.0.0',
            'trace_id': getattr(g, 'trace_id', None),
            **kwargs
        }
        self.logger.info(json.dumps(log_entry))

logger = StructuredLogger()

@app.before_request
def before_request():
    g.trace_id = request.headers.get('X-Trace-Id', str(uuid.uuid4()))

@app.route('/users/<user_id>')
def get_user(user_id):
    logger.log('INFO', 'Fetching user', user_id=user_id, endpoint='/users')
    # Application logic here
    return {'user_id': user_id}
```

**Log Analysis Queries**:
```bash
# Query logs with correlation ID
gcloud logging read 'jsonPayload.trace_id="abc-123-def" AND timestamp>="2023-01-01T00:00:00Z"' \
    --format="table(timestamp,jsonPayload.service,jsonPayload.message)"

# Error rate by service
gcloud logging read 'jsonPayload.level="ERROR" AND timestamp>="2023-01-01T00:00:00Z"' \
    --format="value(jsonPayload.service)" | sort | uniq -c
```

---

## 🔹 CI/CD & DevOps

### Q15: How do you implement GitOps with Cloud Build and GKE?

**Answer**: Automated CI/CD pipeline with Git-based deployments.

**Cloud Build Configuration**:
```yaml
# cloudbuild.yaml
steps:
# Build Docker image
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA', '.']

# Push to Container Registry
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA']

# Update Kubernetes manifests
- name: 'gcr.io/cloud-builders/gke-deploy'
  args:
  - run
  - --filename=k8s/
  - --image=gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA
  - --cluster=production-cluster
  - --location=us-central1-a

# Run tests
- name: 'gcr.io/cloud-builders/kubectl'
  args: ['apply', '-f', 'k8s/test-job.yaml']
  env:
  - 'CLOUDSDK_COMPUTE_ZONE=us-central1-a'
  - 'CLOUDSDK_CONTAINER_CLUSTER=production-cluster'

# Security scanning
- name: 'gcr.io/cloud-builders/gcloud'
  args: ['container', 'images', 'scan', 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA']

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'

timeout: '1200s'
```

**GitOps Repository Structure**:
```
gitops-repo/
├── apps/
│   ├── production/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── staging/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
├── infrastructure/
│   ├── gke-cluster.yaml
│   └── networking.yaml
└── scripts/
    └── deploy.sh
```

**Automated Deployment Script**:
```bash
#!/bin/bash
# deploy.sh

set -e

ENVIRONMENT=${1:-staging}
IMAGE_TAG=${2:-latest}

echo "Deploying to $ENVIRONMENT with image tag $IMAGE_TAG"

# Update image tag in kustomization
cd apps/$ENVIRONMENT
kustomize edit set image gcr.io/PROJECT_ID/myapp=gcr.io/PROJECT_ID/myapp:$IMAGE_TAG

# Apply manifests
kubectl apply -k .

# Wait for rollout
kubectl rollout status deployment/myapp -n $ENVIRONMENT

# Run smoke tests
kubectl run smoke-test --image=gcr.io/PROJECT_ID/smoke-test:latest \
    --env="TARGET_URL=http://myapp.$ENVIRONMENT.svc.cluster.local" \
    --restart=Never

# Check test results
kubectl wait --for=condition=complete job/smoke-test --timeout=300s
```

**Multi-Environment Pipeline**:
```yaml
# Cloud Build trigger for different branches
# staging-trigger.yaml
name: staging-deploy
github:
  owner: company
  name: myapp
  push:
    branch: develop
filename: cloudbuild-staging.yaml

# production-trigger.yaml  
name: production-deploy
github:
  owner: company
  name: myapp
  push:
    tag: v.*
filename: cloudbuild-production.yaml
```

---

## 🔹 Cost Management

### Q16: How do you optimize GCP costs for a production workload?

**Answer**: Multi-faceted approach including right-sizing, committed use, and automation.

**Cost Analysis**:
```bash
# Export billing data to BigQuery
bq mk --dataset PROJECT_ID:billing_analysis

# Query top cost drivers
bq query --use_legacy_sql=false '
SELECT
  service.description as service_name,
  SUM(cost) as total_cost,
  SUM(usage.amount) as total_usage
FROM `PROJECT_ID.billing_dataset.gcp_billing_export_v1_BILLING_ACCOUNT_ID`
WHERE _PARTITIONTIME >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY service.description
ORDER BY total_cost DESC
LIMIT 10'

# Analyze compute costs by machine type
bq query --use_legacy_sql=false '
SELECT
  sku.description,
  SUM(cost) as total_cost,
  COUNT(*) as instance_count
FROM `PROJECT_ID.billing_dataset.gcp_billing_export_v1_BILLING_ACCOUNT_ID`
WHERE service.description = "Compute Engine"
  AND _PARTITIONTIME >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY sku.description
ORDER BY total_cost DESC'
```

**Right-sizing Implementation**:
```bash
# Get rightsizing recommendations
gcloud recommender recommendations list \
    --project=PROJECT_ID \
    --recommender=google.compute.instance.MachineTypeRecommender \
    --location=us-central1-a \
    --format="table(name,description,primaryImpact.costProjection.cost.currencyCode,primaryImpact.costProjection.cost.units)"

# Apply recommendations
gcloud compute instances stop INSTANCE_NAME --zone=us-central1-a
gcloud compute instances set-machine-type INSTANCE_NAME \
    --machine-type=e2-standard-2 \
    --zone=us-central1-a
gcloud compute instances start INSTANCE_NAME --zone=us-central1-a
```

**Committed Use Discounts**:
```bash
# Analyze usage patterns
gcloud compute commitments list

# Create commitment
gcloud compute commitments create my-commitment \
    --plan=12-month \
    --region=us-central1 \
    --resources=vcpu=100,memory=400GB

# Monitor commitment utilization
gcloud billing budgets create \
    --billing-account=BILLING_ACCOUNT_ID \
    --display-name="Commitment Utilization" \
    --budget-amount=10000USD \
    --threshold-rules-percent=0.8,0.9,1.0
```

**Automated Cost Optimization**:
```bash
# Cost optimization script
cat > cost-optimizer.sh << 'EOF'
#!/bin/bash

# Stop dev/test instances during off-hours
if [ $(date +%H) -ge 18 ] || [ $(date +%H) -le 8 ]; then
    gcloud compute instances list \
        --filter="labels.environment=development AND status:RUNNING" \
        --format="value(name,zone)" | \
        while read name zone; do
            echo "Stopping dev instance: $name"
            gcloud compute instances stop $name --zone=$zone --async
        done
fi

# Clean up old snapshots
gcloud compute snapshots list \
    --filter="creationTimestamp<$(date -d '7 days ago' --iso-8601)" \
    --format="value(name)" | \
    while read snapshot; do
        echo "Deleting old snapshot: $snapshot"
        gcloud compute snapshots delete $snapshot --quiet
    done

# Identify unused persistent disks
gcloud compute disks list --filter="-users:*" \
    --format="value(name,zone,sizeGb)" | \
    while read name zone size; do
        echo "Unused disk: $name ($size GB) in $zone"
        # Optionally delete after confirmation
        # gcloud compute disks delete $name --zone=$zone --quiet
    done
EOF

# Schedule with Cloud Scheduler
gcloud scheduler jobs create http cost-optimizer-job \
    --schedule="0 */6 * * *" \
    --uri="https://REGION-PROJECT_ID.cloudfunctions.net/cost-optimizer" \
    --http-method=POST
```

**Preemptible/Spot Instance Strategy**:
```bash
# Create mixed instance group template
gcloud compute instance-templates create mixed-template \
    --machine-type=e2-standard-4 \
    --provisioning-model=SPOT \
    --instance-termination-action=STOP \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud

# Implement graceful shutdown
cat > graceful-shutdown.sh << 'EOF'
#!/bin/bash
# Handle preemption gracefully

# Check for preemption notice
if curl -H "Metadata-Flavor: Google" \
   http://metadata.google.internal/computeMetadata/v1/instance/preempted 2>/dev/null | grep -q TRUE; then
    echo "Instance preempted, shutting down gracefully"
    
    # Stop application services
    systemctl stop myapp
    
    # Save state to persistent storage
    gsutil cp /tmp/app-state gs://backup-bucket/state-$(hostname)-$(date +%s)
    
    # Signal completion
    exit 0
fi
EOF
```

This comprehensive guide covers the most important GCP concepts and scenarios that senior DevOps engineers encounter in interviews and production environments. Each answer includes practical implementations, best practices, and real-world considerations.
# GCP IAM and Kubernetes (GKE) Integration Guide

## Table of Contents
1. [IAM Fundamentals](#iam-fundamentals)
2. [IAM Integration with Kubernetes (GKE)](#iam-integration-with-kubernetes-gke)
3. [Service Accounts](#service-accounts)
4. [User Identity Management](#user-identity-management)
5. [Cloud Storage ACL](#cloud-storage-acl)
6. [Resource Hierarchy](#resource-hierarchy)
7. [Organization Policy Service](#organization-policy-service)
8. [Billing and Budget Alerts](#billing-and-budget-alerts)
9. [Best Practices](#best-practices)

## IAM Fundamentals

### What is IAM?
Identity and Access Management (IAM) is Google Cloud's unified system for managing access to GCP resources. It follows the principle of "who can do what on which resource."

### IAM Components

#### 1. IAM Role, Policy, and Member

**Member**: An identity that can be granted access to a resource
- Google Account (user@gmail.com)
- Service Account (service@project.iam.gserviceaccount.com)
- Google Group (group@googlegroups.com)
- Google Workspace domain (example.com)
- Cloud Identity domain

**Role**: A collection of permissions that define what actions can be performed
**Policy**: Binds members to roles for specific resources

```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:alice@example.com",
        "serviceAccount:my-service@project.iam.gserviceaccount.com"
      ]
    }
  ]
}
```

#### 2. Role Types

**Primitive Roles** (Legacy - avoid in production):
- Owner: Full access to all resources
- Editor: Read/write access to most resources
- Viewer: Read-only access

**Predefined Roles** (Recommended):
- Created and maintained by Google
- Follow principle of least privilege
- Examples: `roles/compute.instanceAdmin`, `roles/storage.objectViewer`

**Custom Roles**:
- User-defined roles with specific permissions
- Can be created at organization or project level

```yaml
# Custom role example
title: "Custom Storage Role"
description: "Custom role for storage operations"
stage: "GA"
includedPermissions:
- storage.objects.get
- storage.objects.list
- storage.buckets.get
```

#### 3. Conditions in Policy

IAM Conditions allow you to grant access based on attributes like time, IP address, or resource attributes.

```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": ["user:alice@example.com"],
      "condition": {
        "title": "Time-based access",
        "description": "Access only during business hours",
        "expression": "request.time.getHours() >= 9 && request.time.getHours() < 17"
      }
    }
  ]
}
```

Common condition expressions:
- Time-based: `request.time.getHours() < 12`
- IP-based: `inIpRange(origin.ip, '203.0.113.0/24')`
- Resource-based: `resource.name.startsWith('projects/my-project/buckets/sensitive-')`

## Service Accounts

### What are Service Accounts?

Service accounts are special Google accounts that represent applications or compute workloads, not individual users. They're used for server-to-server interactions.

### Types of Service Accounts

1. **User-managed Service Accounts**
   - Created and managed by users
   - Can be used across projects
   - Maximum 100 per project

2. **Default Service Accounts**
   - Automatically created with certain services
   - Compute Engine default service account
   - App Engine default service account

3. **Google-managed Service Accounts**
   - Created and managed by Google services
   - Used internally by Google services

### Creating and Managing Service Accounts

```bash
# Create service account
gcloud iam service-accounts create my-service-account \
    --description="My service account" \
    --display-name="My Service Account"

# Grant roles to service account
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:my-service-account@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# Create and download key
gcloud iam service-accounts keys create ~/key.json \
    --iam-account=my-service-account@PROJECT_ID.iam.gserviceaccount.com
```

### Grant Users Access to Service Account

The "Grant users access to this service account" option allows users to:
- Impersonate the service account
- Create access tokens for the service account
- Use the service account in their applications

```bash
# Grant user ability to impersonate service account
gcloud iam service-accounts add-iam-policy-binding \
    my-service-account@PROJECT_ID.iam.gserviceaccount.com \
    --member="user:alice@example.com" \
    --role="roles/iam.serviceAccountUser"

# Grant token creator role
gcloud iam service-accounts add-iam-policy-binding \
    my-service-account@PROJECT_ID.iam.gserviceaccount.com \
    --member="user:alice@example.com" \
    --role="roles/iam.serviceAccountTokenCreator"
```

### Using Service Accounts Outside GCP Resources

#### Short-lived Tokens (Recommended)
```bash
# Generate access token (1 hour expiry)
gcloud auth print-access-token --impersonate-service-account=SERVICE_ACCOUNT_EMAIL

# Using with curl
TOKEN=$(gcloud auth print-access-token --impersonate-service-account=SERVICE_ACCOUNT_EMAIL)
curl -H "Authorization: Bearer $TOKEN" https://storage.googleapis.com/storage/v1/b
```

#### Long-lived Service Account Keys
```bash
# Create key file (not recommended for production)
gcloud iam service-accounts keys create key.json \
    --iam-account=SERVICE_ACCOUNT_EMAIL

# Use in application
export GOOGLE_APPLICATION_CREDENTIALS="path/to/key.json"
```

## IAM Integration with Kubernetes (GKE)

### Key Concepts

#### 1. Workload Identity
Workload Identity allows Kubernetes service accounts to act as Google service accounts, eliminating the need to store service account keys in pods.

#### 2. Google Service Account (GSA)
Google Cloud IAM service account that has permissions to GCP resources.

#### 3. Kubernetes Service Account (KSA)
Kubernetes-native service account that pods use for authentication.

### Architecture

```
Pod → KSA → GSA → GCP Resources
```

1. Pod uses Kubernetes Service Account
2. KSA is bound to Google Service Account via Workload Identity
3. GSA has IAM permissions to access GCP resources

### Use Cases

- **Secure Access**: Pods accessing Cloud Storage, BigQuery, or other GCP services
- **CI/CD Pipelines**: Automated deployments requiring GCP resource access
- **Monitoring**: Applications sending metrics to Cloud Monitoring
- **Secrets Management**: Accessing Secret Manager from pods

### Required Configurations

#### 1. Enable Workload Identity on GKE Cluster

```bash
# Create cluster with Workload Identity
gcloud container clusters create my-cluster \
    --workload-pool=PROJECT_ID.svc.id.goog \
    --zone=us-central1-a

# Enable on existing cluster
gcloud container clusters update my-cluster \
    --workload-pool=PROJECT_ID.svc.id.goog \
    --zone=us-central1-a
```

#### 2. Create Google Service Account

```bash
# Create GSA
gcloud iam service-accounts create gsa-name \
    --description="Service account for GKE workload" \
    --display-name="GKE Workload SA"

# Grant necessary permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:gsa-name@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"
```

#### 3. Create Kubernetes Service Account

```yaml
# ksa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ksa-name
  namespace: default
  annotations:
    iam.gke.io/gcp-service-account: gsa-name@PROJECT_ID.iam.gserviceaccount.com
```

#### 4. Bind KSA to GSA

```bash
# Allow KSA to impersonate GSA
gcloud iam service-accounts add-iam-policy-binding \
    gsa-name@PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"
```

### Step-by-Step Setup Guide

#### Step 1: Prepare Environment

```bash
# Set variables
export PROJECT_ID="your-project-id"
export CLUSTER_NAME="my-gke-cluster"
export ZONE="us-central1-a"
export GSA_NAME="my-gsa"
export KSA_NAME="my-ksa"
export NAMESPACE="default"

# Enable APIs
gcloud services enable container.googleapis.com
gcloud services enable iam.googleapis.com
```

#### Step 2: Create GKE Cluster

```bash
gcloud container clusters create $CLUSTER_NAME \
    --zone=$ZONE \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --enable-ip-alias \
    --num-nodes=3
```

#### Step 3: Create Google Service Account

```bash
gcloud iam service-accounts create $GSA_NAME \
    --description="Service account for GKE workload" \
    --display-name="GKE Workload SA"

# Grant Cloud Storage permissions
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"
```

#### Step 4: Create Kubernetes Resources

```yaml
# workload-identity-setup.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-ksa
  namespace: default
  annotations:
    iam.gke.io/gcp-service-account: my-gsa@PROJECT_ID.iam.gserviceaccount.com
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-identity-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: workload-identity-test
  template:
    metadata:
      labels:
        app: workload-identity-test
    spec:
      serviceAccountName: my-ksa
      containers:
      - name: test-container
        image: google/cloud-sdk:slim
        command: ["sleep", "infinity"]
```

#### Step 5: Bind Service Accounts

```bash
gcloud iam service-accounts add-iam-policy-binding \
    $GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]"
```

#### Step 6: Deploy and Test

```bash
# Apply manifests
kubectl apply -f workload-identity-setup.yaml

# Test access
kubectl exec -it deployment/workload-identity-test -- gcloud auth list
kubectl exec -it deployment/workload-identity-test -- gsutil ls gs://your-bucket
```

### Helm Chart Example

```yaml
# values.yaml
serviceAccount:
  create: true
  name: "my-app-ksa"
  annotations:
    iam.gke.io/gcp-service-account: "my-app-gsa@PROJECT_ID.iam.gserviceaccount.com"

deployment:
  image: "my-app:latest"
  serviceAccountName: "my-app-ksa"
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.deployment.name }}
spec:
  template:
    spec:
      serviceAccountName: {{ .Values.serviceAccount.name }}
      containers:
      - name: app
        image: {{ .Values.deployment.image }}
```

## User Identity Management

### Using Google Workspace

Google Workspace provides centralized identity management for organizations.

#### Setup Steps:

1. **Configure Google Workspace Domain**
```bash
# Add domain to GCP organization
gcloud organizations add-iam-policy-binding ORGANIZATION_ID \
    --member="domain:example.com" \
    --role="roles/resourcemanager.organizationViewer"
```

2. **Sync Users and Groups**
```bash
# Grant domain users access to project
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="domain:example.com" \
    --role="roles/viewer"

# Grant specific group access
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="group:developers@example.com" \
    --role="roles/editor"
```

### Using Federated Cloud Identity

Federate external identity providers (SAML, OIDC) with Google Cloud.

#### SAML Federation Example:

```bash
# Create workforce pool
gcloud iam workforce-pools create my-workforce-pool \
    --organization=ORGANIZATION_ID \
    --location=global \
    --display-name="My Workforce Pool"

# Create SAML provider
gcloud iam workforce-pools providers create-saml my-saml-provider \
    --workforce-pool=my-workforce-pool \
    --location=global \
    --idp-metadata-path=metadata.xml \
    --attribute-mapping="google.subject=assertion.subject"
```

#### OIDC Federation Example:

```bash
# Create workload identity pool
gcloud iam workload-identity-pools create my-pool \
    --project=PROJECT_ID \
    --location=global \
    --display-name="My Pool"

# Create OIDC provider
gcloud iam workload-identity-pools providers create-oidc my-oidc-provider \
    --project=PROJECT_ID \
    --location=global \
    --workload-identity-pool=my-pool \
    --issuer-uri="https://accounts.google.com" \
    --allowed-audiences="my-audience"
```

## Cloud Storage ACL

### ACL Types

#### 1. Uniform Bucket-level Access (Recommended)
Uses only IAM for access control.

```bash
# Enable uniform bucket-level access
gsutil uniformbucketlevelaccess set on gs://my-bucket

# Grant access via IAM
gsutil iam ch user:alice@example.com:objectViewer gs://my-bucket
```

#### 2. Fine-grained ACLs
Object-level and bucket-level ACLs.

```bash
# Set bucket ACL
gsutil acl set private gs://my-bucket

# Set object ACL
gsutil acl ch -u alice@example.com:READ gs://my-bucket/object.txt

# Predefined ACLs
gsutil acl set public-read gs://my-bucket/public-object.txt
```

#### 3. ACL Roles
- **READER**: Read access to object/bucket
- **WRITER**: Read/write access to objects
- **OWNER**: Full control over object/bucket

### Signed URLs

Provide time-limited access to Cloud Storage objects without requiring authentication.

```python
# Python example
from google.cloud import storage
from datetime import datetime, timedelta

def generate_signed_url(bucket_name, blob_name):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    
    url = blob.generate_signed_url(
        version="v4",
        expiration=datetime.utcnow() + timedelta(hours=1),
        method="GET"
    )
    return url
```

```bash
# Using gsutil
gsutil signurl -d 1h service-account-key.json gs://my-bucket/object.txt
```

## Resource Hierarchy

### GCP Resource Hierarchy Structure

```
Organization
├── Folder (Optional)
│   ├── Project A
│   │   ├── Compute Engine
│   │   ├── Cloud Storage
│   │   └── GKE Clusters
│   └── Project B
└── Project C
```

### Hierarchy Benefits

1. **Inheritance**: IAM policies inherit down the hierarchy
2. **Organization**: Logical grouping of resources
3. **Billing**: Centralized billing management
4. **Compliance**: Organization-wide policies

### Managing Hierarchy

```bash
# Create folder
gcloud resource-manager folders create \
    --display-name="Development" \
    --organization=ORGANIZATION_ID

# Move project to folder
gcloud projects move PROJECT_ID --folder=FOLDER_ID

# Set IAM policy at organization level
gcloud organizations add-iam-policy-binding ORGANIZATION_ID \
    --member="group:admins@example.com" \
    --role="roles/resourcemanager.organizationAdmin"
```

## Organization Policy Service

Organization policies provide centralized control over resources across the organization.

### Common Organization Policies

#### 1. Restrict VM External IP
```yaml
# org-policy-no-external-ip.yaml
constraint: constraints/compute.vmExternalIpAccess
listPolicy:
  deniedValues:
  - "*"
```

#### 2. Allowed Locations
```yaml
# org-policy-allowed-locations.yaml
constraint: constraints/gcp.resourceLocations
listPolicy:
  allowedValues:
  - "us-central1"
  - "us-east1"
```

#### 3. Require OS Login
```yaml
# org-policy-os-login.yaml
constraint: constraints/compute.requireOsLogin
booleanPolicy:
  enforced: true
```

### Applying Organization Policies

```bash
# Set policy at organization level
gcloud resource-manager org-policies set-policy org-policy-no-external-ip.yaml \
    --organization=ORGANIZATION_ID

# Set policy at project level
gcloud resource-manager org-policies set-policy org-policy-allowed-locations.yaml \
    --project=PROJECT_ID

# List effective policies
gcloud resource-manager org-policies list --organization=ORGANIZATION_ID
```

## Billing and Budget Alerts

### Setting Up Billing Budgets

#### 1. Create Budget via CLI

```bash
# Create budget
gcloud billing budgets create \
    --billing-account=BILLING_ACCOUNT_ID \
    --display-name="Monthly Budget" \
    --budget-amount=1000USD \
    --threshold-rule=percent=50,basis=CURRENT_SPEND \
    --threshold-rule=percent=90,basis=CURRENT_SPEND \
    --threshold-rule=percent=100,basis=FORECASTED_SPEND
```

#### 2. Budget Configuration JSON

```json
{
  "displayName": "Monthly Project Budget",
  "budgetFilter": {
    "projects": ["projects/PROJECT_ID"],
    "creditTypesTreatment": "INCLUDE_ALL_CREDITS"
  },
  "amount": {
    "specifiedAmount": {
      "currencyCode": "USD",
      "units": "1000"
    }
  },
  "thresholdRules": [
    {
      "thresholdPercent": 0.5,
      "spendBasis": "CURRENT_SPEND"
    },
    {
      "thresholdPercent": 0.9,
      "spendBasis": "CURRENT_SPEND"
    }
  ],
  "notificationsRule": {
    "pubsubTopic": "projects/PROJECT_ID/topics/billing-alerts",
    "schemaVersion": "1.0"
  }
}
```

### Billing Export

#### 1. BigQuery Export

```bash
# Enable billing export to BigQuery
gcloud billing accounts get-iam-policy BILLING_ACCOUNT_ID

# Set up export (via Console or API)
# Creates dataset: billing_export
# Table: gcp_billing_export_v1_BILLING_ACCOUNT_ID
```

#### 2. Cloud Storage Export

```bash
# Export to Cloud Storage bucket
gsutil mb gs://billing-export-bucket

# Configure export in Cloud Console:
# Billing → Billing Export → File Export
```

### Budget Alert Automation

```python
# Cloud Function for budget alerts
import json
from google.cloud import compute_v1

def budget_alert_handler(event, context):
    """Handle budget alert and take action"""
    pubsub_message = json.loads(base64.b64decode(event['data']).decode('utf-8'))
    
    budget_amount = pubsub_message['budgetAmount']
    cost_amount = pubsub_message['costAmount']
    
    if cost_amount >= budget_amount * 0.9:
        # Stop all compute instances
        compute_client = compute_v1.InstancesClient()
        # Implementation to stop instances
        pass
```

## Best Practices

### IAM Best Practices

#### 1. Principle of Least Privilege
```bash
# Bad: Granting broad permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="user:alice@example.com" \
    --role="roles/editor"

# Good: Granting specific permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="user:alice@example.com" \
    --role="roles/storage.objectViewer"
```

#### 2. Use Groups Instead of Individual Users
```bash
# Create group and assign roles to group
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="group:developers@example.com" \
    --role="roles/compute.instanceAdmin"
```

#### 3. Regular Access Reviews
```bash
# Audit IAM policies
gcloud projects get-iam-policy PROJECT_ID --format=json > iam-policy.json

# Use Cloud Asset Inventory for organization-wide audit
gcloud asset search-all-iam-policies --scope=organizations/ORGANIZATION_ID
```

#### 4. Service Account Key Management
```bash
# Avoid long-lived keys, use Workload Identity instead
# If keys are necessary, rotate regularly
gcloud iam service-accounts keys create new-key.json \
    --iam-account=SERVICE_ACCOUNT_EMAIL

gcloud iam service-accounts keys delete KEY_ID \
    --iam-account=SERVICE_ACCOUNT_EMAIL
```

### GKE Security Best Practices

#### 1. Enable Workload Identity
```bash
# Always use Workload Identity instead of service account keys
gcloud container clusters update CLUSTER_NAME \
    --workload-pool=PROJECT_ID.svc.id.goog
```

#### 2. Use Binary Authorization
```bash
# Enable Binary Authorization
gcloud container binauthz policy import policy.yaml
```

#### 3. Network Policies
```yaml
# Network policy example
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### 4. Pod Security Standards
```yaml
# Pod Security Policy
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Cost Optimization

#### 1. Right-sizing Resources
```bash
# Use appropriate machine types
gcloud container clusters create cost-optimized-cluster \
    --machine-type=e2-medium \
    --enable-autoscaling \
    --min-nodes=1 \
    --max-nodes=10
```

#### 2. Preemptible Instances
```bash
# Use preemptible nodes for non-critical workloads
gcloud container node-pools create preemptible-pool \
    --cluster=CLUSTER_NAME \
    --preemptible \
    --machine-type=e2-medium
```

#### 3. Monitoring and Alerting
```yaml
# Resource quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

### Security Best Practices

#### 1. Enable Audit Logging
```yaml
# Cloud Logging configuration
auditConfigs:
- service: allServices
  auditLogConfigs:
  - logType: ADMIN_READ
  - logType: DATA_READ
  - logType: DATA_WRITE
```

#### 2. Use Private Clusters
```bash
gcloud container clusters create private-cluster \
    --enable-private-nodes \
    --master-ipv4-cidr-block 172.16.0.0/28 \
    --enable-ip-alias
```

#### 3. Regular Security Scanning
```bash
# Enable vulnerability scanning
gcloud container images scan IMAGE_URL
```

This comprehensive guide covers IAM integration with Kubernetes (GKE), including all the concepts you requested. The examples provided are practical and can be adapted to your specific use cases. Remember to always follow the principle of least privilege and regularly review your IAM policies for security and cost optimization.
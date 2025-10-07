# 🔐 GCP Cloud Security Comprehensive Guide for Senior DevOps Engineers
*Advanced Cloud Security Architecture and Implementation*

## Table of Contents
1. [GCP Security Model & Shared Responsibility](#gcp-security-model--shared-responsibility)
2. [Identity & Access Management (IAM)](#identity--access-management-iam)
3. [Network Security Architecture](#network-security-architecture)
4. [Data Protection & Encryption](#data-protection--encryption)
5. [Compute Security & Hardening](#compute-security--hardening)
6. [Container & Kubernetes Security](#container--kubernetes-security)
7. [Application Security & DevSecOps](#application-security--devsecops)
8. [Monitoring, Logging & Incident Response](#monitoring-logging--incident-response)
9. [Compliance & Governance](#compliance--governance)
10. [Security Automation & Infrastructure as Code](#security-automation--infrastructure-as-code)
11. [Threat Detection & Response](#threat-detection--response)
12. [Security Best Practices & Architecture Patterns](#security-best-practices--architecture-patterns)

---

## 🔹 GCP Security Model & Shared Responsibility

### Google's Security Infrastructure

**Defense in Depth Architecture:**
Google implements multiple layers of security controls across their infrastructure:

**Physical Security:**
- **Data Center Security**: Biometric access controls, security cameras, 24/7 guards
- **Hardware Security**: Custom-designed servers with hardware security modules
- **Supply Chain Security**: Secure hardware procurement and verification
- **Decommissioning**: Secure data destruction and hardware disposal

**Infrastructure Security:**
- **Network Security**: Private global network with encryption in transit
- **Service Identity**: Mutual TLS authentication between services
- **Isolation**: Multi-tenant isolation using hardware and software controls
- **Encryption**: Data encrypted at rest and in transit by default

**Platform Security:**
- **Boot Integrity**: Verified boot process with cryptographic signatures
- **Service Deployment**: Automated security scanning and deployment
- **Vulnerability Management**: Continuous security monitoring and patching
- **Access Controls**: Strict employee access controls and monitoring

### Shared Responsibility Model

**Google's Responsibilities:**
- **Infrastructure Security**: Physical security, network security, platform hardening
- **Service Security**: API security, service-to-service authentication
- **Data Encryption**: Encryption at rest and in transit
- **Compliance**: Infrastructure compliance certifications

**Customer Responsibilities:**
- **Identity Management**: User authentication and authorization
- **Data Classification**: Sensitive data identification and protection
- **Application Security**: Secure coding practices and vulnerability management
- **Configuration Management**: Secure service configuration and access controls

**Shared Responsibilities:**
- **Network Controls**: Firewall rules, VPC configuration
- **Operating System**: Patching and hardening (for Compute Engine)
- **Access Management**: Service account and user access controls
- **Monitoring**: Security monitoring and incident response

### Security by Default

**Automatic Security Features:**
- **Encryption**: All data encrypted at rest and in transit by default
- **Network Isolation**: VPC provides network isolation by default
- **Identity Verification**: Strong authentication required for all access
- **Audit Logging**: Comprehensive audit trails for all API calls

**Security-First Design:**
- **Least Privilege**: Minimal permissions granted by default
- **Zero Trust**: No implicit trust based on network location
- **Continuous Monitoring**: Real-time security monitoring and alerting
- **Automated Response**: Automated threat detection and response

---

## 🔹 Identity & Access Management (IAM)

### IAM Fundamentals

**IAM Components:**
- **Members**: Who can access resources (users, groups, service accounts)
- **Roles**: What permissions are granted (collections of permissions)
- **Resources**: What can be accessed (projects, folders, organizations)
- **Policies**: Binding of members to roles on resources

**IAM Hierarchy:**
```
Organization
├── Folders
│   ├── Projects
│   │   ├── Resources
│   │   └── Service Accounts
│   └── Subfolders
└── Billing Accounts
```

**Permission Inheritance:**
Permissions granted at higher levels automatically apply to lower levels:
- Organization-level permissions apply to all folders and projects
- Folder-level permissions apply to all contained projects
- Project-level permissions apply to all contained resources

### Advanced IAM Concepts

**Predefined Roles:**
Google-managed roles with curated permissions:
```bash
# Viewer role - read-only access
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:user@example.com" \
  --role="roles/viewer"

# Editor role - modify access (no IAM changes)
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:user@example.com" \
  --role="roles/editor"

# Owner role - full access including IAM
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:user@example.com" \
  --role="roles/owner"
```

**Custom Roles:**
Organization-specific roles with tailored permissions:
```yaml
# custom-role.yaml
title: "Custom Compute Admin"
description: "Custom role for compute administration"
stage: "GA"
includedPermissions:
- compute.instances.create
- compute.instances.delete
- compute.instances.start
- compute.instances.stop
- compute.disks.create
- compute.networks.use
```

```bash
# Create custom role
gcloud iam roles create customComputeAdmin \
  --organization=ORGANIZATION_ID \
  --file=custom-role.yaml
```

**Service Accounts:**
Non-human identities for applications and services:

**Service Account Types:**
- **User-managed**: Created and managed by users
- **Google-managed**: Created and managed by Google services
- **Default**: Automatically created for some services

**Service Account Best Practices:**
```bash
# Create service account
gcloud iam service-accounts create my-service-account \
  --display-name="My Service Account" \
  --description="Service account for application X"

# Generate and download key
gcloud iam service-accounts keys create key.json \
  --iam-account=my-service-account@PROJECT_ID.iam.gserviceaccount.com

# Use workload identity (recommended)
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]" \
  GSA_NAME@PROJECT_ID.iam.gserviceaccount.com
```

### Conditional IAM

**IAM Conditions:**
Fine-grained access control based on attributes:

**Condition Attributes:**
- **Time-based**: Date, time, day of week
- **Resource-based**: Resource name, type, labels
- **Request-based**: IP address, access level
- **Environment-based**: Device policy, location

**Example Conditions:**
```bash
# Time-based access (business hours only)
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:user@example.com" \
  --role="roles/compute.instanceAdmin" \
  --condition='expression=request.time.getHours() >= 9 && request.time.getHours() < 17,title=Business Hours Only,description=Access only during business hours'

# IP-based access
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:user@example.com" \
  --role="roles/storage.admin" \
  --condition='expression=inIpRange(origin.ip, "203.0.113.0/24"),title=Office IP Only,description=Access only from office IP range'

# Resource-based access
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:user@example.com" \
  --role="roles/compute.instanceAdmin" \
  --condition='expression=resource.name.startsWith("projects/PROJECT_ID/zones/us-central1-a/"),title=Zone Specific,description=Access only to us-central1-a resources'
```

### Organization Policies

**Policy Constraints:**
Organization-wide policies that restrict resource configuration:

**Common Policy Constraints:**
```bash
# Restrict VM external IP
gcloud resource-manager org-policies set-policy \
  --organization=ORGANIZATION_ID \
  constraint_config.yaml

# constraint_config.yaml
constraint: constraints/compute.vmExternalIpAccess
booleanPolicy:
  enforced: true

# Restrict allowed machine types
constraint: constraints/compute.vmCanIpForward
listPolicy:
  allowedValues:
  - "projects/PROJECT_ID/zones/*/machineTypes/n1-standard-1"
  - "projects/PROJECT_ID/zones/*/machineTypes/n1-standard-2"

# Require OS Login
constraint: constraints/compute.requireOsLogin
booleanPolicy:
  enforced: true
```

---

## 🔹 Network Security Architecture

### VPC Security Design

**VPC Network Isolation:**
Virtual Private Cloud provides network-level isolation:

**VPC Components:**
- **Subnets**: Regional IP address ranges
- **Routes**: Traffic routing rules
- **Firewall Rules**: Traffic filtering rules
- **VPC Peering**: Connect VPCs securely

**Network Segmentation Strategies:**
```bash
# Create VPC with custom subnets
gcloud compute networks create secure-vpc --subnet-mode=custom

# Create subnets for different tiers
gcloud compute networks subnets create web-subnet \
  --network=secure-vpc \
  --range=10.1.1.0/24 \
  --region=us-central1

gcloud compute networks subnets create app-subnet \
  --network=secure-vpc \
  --range=10.1.2.0/24 \
  --region=us-central1

gcloud compute networks subnets create db-subnet \
  --network=secure-vpc \
  --range=10.1.3.0/24 \
  --region=us-central1
```

### Firewall Rules and Security

**Hierarchical Firewall Policies:**
Organization-level firewall rules that apply across projects:

**Firewall Rule Types:**
- **VPC Firewall Rules**: Project-level rules
- **Hierarchical Firewall Policies**: Organization/folder-level rules
- **Global Network Firewall Policies**: Cross-VPC rules

**Advanced Firewall Configuration:**
```bash
# Create hierarchical firewall policy
gcloud compute network-firewall-policies create secure-policy \
  --global \
  --description="Organization security policy"

# Add rules to policy
gcloud compute network-firewall-policies rules create 1000 \
  --firewall-policy=secure-policy \
  --global \
  --action=deny \
  --direction=ingress \
  --src-ip-ranges=0.0.0.0/0 \
  --layer4-configs=tcp:22 \
  --description="Deny SSH from internet"

# Allow SSH from specific IP ranges
gcloud compute network-firewall-policies rules create 1100 \
  --firewall-policy=secure-policy \
  --global \
  --action=allow \
  --direction=ingress \
  --src-ip-ranges=203.0.113.0/24 \
  --layer4-configs=tcp:22 \
  --description="Allow SSH from office"

# Associate policy with organization
gcloud compute network-firewall-policies associations create \
  --firewall-policy=secure-policy \
  --global \
  --organization=ORGANIZATION_ID
```

**Network Tags and Service Accounts:**
```bash
# Create firewall rule using network tags
gcloud compute firewall-rules create allow-web-traffic \
  --network=secure-vpc \
  --action=allow \
  --direction=ingress \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server

# Create firewall rule using service accounts
gcloud compute firewall-rules create allow-app-to-db \
  --network=secure-vpc \
  --action=allow \
  --direction=ingress \
  --rules=tcp:3306 \
  --source-service-accounts=app-service-account@PROJECT_ID.iam.gserviceaccount.com \
  --target-service-accounts=db-service-account@PROJECT_ID.iam.gserviceaccount.com
```

### Private Google Access and Service Controls

**Private Google Access:**
Access Google services without external IP addresses:

**Configuration:**
```bash
# Enable Private Google Access on subnet
gcloud compute networks subnets update SUBNET_NAME \
  --region=REGION \
  --enable-private-ip-google-access

# Create custom route for restricted Google APIs
gcloud compute routes create restricted-google-apis \
  --network=VPC_NAME \
  --destination-range=199.36.153.8/30 \
  --next-hop-gateway=default-internet-gateway \
  --priority=1000
```

**VPC Service Controls:**
Create security perimeters around Google Cloud resources:

**Service Perimeter Configuration:**
```bash
# Create access policy
gcloud access-context-manager policies create \
  --organization=ORGANIZATION_ID \
  --title="Production Access Policy"

# Create service perimeter
gcloud access-context-manager perimeters create production-perimeter \
  --policy=POLICY_ID \
  --title="Production Perimeter" \
  --resources=projects/PROJECT_ID \
  --restricted-services=storage.googleapis.com,bigquery.googleapis.com \
  --perimeter-type=regular

# Create access level
gcloud access-context-manager levels create office-access \
  --policy=POLICY_ID \
  --title="Office Access" \
  --basic-level-spec=office-access.yaml
```

```yaml
# office-access.yaml
conditions:
- ipSubnetworks:
  - "203.0.113.0/24"
- devicePolicy:
    requireScreenlock: true
    requireAdminApproval: false
    requireCorpOwned: true
```

### Cloud NAT and Load Balancing Security

**Cloud NAT Security:**
Secure outbound internet access for private instances:

```bash
# Create Cloud Router
gcloud compute routers create nat-router \
  --network=secure-vpc \
  --region=us-central1

# Create Cloud NAT
gcloud compute routers nats create secure-nat \
  --router=nat-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips \
  --enable-logging \
  --log-filter=ALL
```

**Load Balancer Security:**
```bash
# Create SSL certificate
gcloud compute ssl-certificates create secure-cert \
  --domains=example.com,www.example.com \
  --global

# Create security policy
gcloud compute security-policies create web-security-policy \
  --description="Web application security policy"

# Add rate limiting rule
gcloud compute security-policies rules create 1000 \
  --security-policy=web-security-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP

# Create backend service with security policy
gcloud compute backend-services create secure-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=web-health-check \
  --security-policy=web-security-policy \
  --global
```

---

## 🔹 Data Protection & Encryption

### Encryption at Rest

**Google-managed Encryption:**
Default encryption for all data at rest:

**Encryption Layers:**
- **Application Layer**: Application-level encryption
- **Infrastructure Layer**: Google's infrastructure encryption
- **Device Layer**: Hardware-level encryption

**Customer-managed Encryption Keys (CMEK):**
Customer control over encryption keys:

```bash
# Create KMS key ring
gcloud kms keyrings create secure-keyring \
  --location=global

# Create encryption key
gcloud kms keys create secure-key \
  --location=global \
  --keyring=secure-keyring \
  --purpose=encryption

# Grant service account access to key
gcloud kms keys add-iam-policy-binding secure-key \
  --location=global \
  --keyring=secure-keyring \
  --member=serviceAccount:service-PROJECT_NUMBER@compute-system.iam.gserviceaccount.com \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter

# Create encrypted disk
gcloud compute disks create secure-disk \
  --size=100GB \
  --zone=us-central1-a \
  --kms-key=projects/PROJECT_ID/locations/global/keyRings/secure-keyring/cryptoKeys/secure-key
```

**Customer-supplied Encryption Keys (CSEK):**
Customer-provided encryption keys:

```bash
# Generate encryption key
openssl rand -base64 32 > encryption-key.txt

# Create disk with CSEK
gcloud compute disks create csek-disk \
  --size=100GB \
  --zone=us-central1-a \
  --csek-key-file=encryption-key.txt
```

### Encryption in Transit

**TLS/SSL Configuration:**
Secure communication channels:

**Application Load Balancer SSL:**
```bash
# Create managed SSL certificate
gcloud compute ssl-certificates create managed-cert \
  --domains=example.com \
  --global

# Create HTTPS load balancer
gcloud compute url-maps create secure-lb \
  --default-service=backend-service

gcloud compute target-https-proxies create secure-proxy \
  --url-map=secure-lb \
  --ssl-certificates=managed-cert

gcloud compute global-forwarding-rules create secure-forwarding-rule \
  --target-https-proxy=secure-proxy \
  --global \
  --ports=443
```

**Private Service Connect:**
Secure connectivity to Google services:

```bash
# Create Private Service Connect endpoint
gcloud compute addresses create psc-endpoint-ip \
  --region=us-central1 \
  --subnet=private-subnet

gcloud compute forwarding-rules create psc-endpoint \
  --region=us-central1 \
  --network=secure-vpc \
  --address=psc-endpoint-ip \
  --target-service-attachment=projects/PROJECT_ID/regions/us-central1/serviceAttachments/SERVICE_ATTACHMENT
```

### Key Management Service (KMS)

**KMS Architecture:**
Centralized key management service:

**Key Hierarchy:**
```
Organization
├── Key Rings (Regional/Global)
│   ├── Crypto Keys
│   │   ├── Key Versions
│   │   └── Key Metadata
│   └── Import Jobs
└── IAM Policies
```

**Advanced KMS Operations:**
```bash
# Create key with rotation
gcloud kms keys create auto-rotate-key \
  --location=global \
  --keyring=secure-keyring \
  --purpose=encryption \
  --rotation-period=90d \
  --next-rotation-time=2024-01-01T00:00:00Z

# Encrypt data
echo "sensitive data" | gcloud kms encrypt \
  --key=secure-key \
  --keyring=secure-keyring \
  --location=global \
  --plaintext-file=- \
  --ciphertext-file=encrypted-data.bin

# Decrypt data
gcloud kms decrypt \
  --key=secure-key \
  --keyring=secure-keyring \
  --location=global \
  --ciphertext-file=encrypted-data.bin \
  --plaintext-file=-

# Create import job for external keys
gcloud kms import-jobs create import-job-1 \
  --location=global \
  --keyring=secure-keyring \
  --import-method=rsa-oaep-3072-sha1-aes-256 \
  --protection-level=software
```

**Envelope Encryption:**
Two-tier encryption system:

**Implementation Pattern:**
1. **Data Encryption Key (DEK)**: Encrypts actual data
2. **Key Encryption Key (KEK)**: Encrypts the DEK
3. **Storage**: Encrypted data + encrypted DEK stored together
4. **Decryption**: KEK decrypts DEK, DEK decrypts data

### Secret Management

**Secret Manager:**
Centralized secret storage and management:

```bash
# Create secret
echo "database-password" | gcloud secrets create db-password \
  --data-file=-

# Create secret version
echo "new-database-password" | gcloud secrets versions add db-password \
  --data-file=-

# Access secret
gcloud secrets versions access latest --secret=db-password

# Grant access to service account
gcloud secrets add-iam-policy-binding db-password \
  --member=serviceAccount:app@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

**Secret Manager Integration:**
```python
# Python application integration
from google.cloud import secretmanager

def get_secret(project_id, secret_id, version_id="latest"):
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version_id}"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

# Usage
db_password = get_secret("my-project", "db-password")
```

---

## 🔹 Compute Security & Hardening

### Compute Engine Security

**OS Login:**
Centralized SSH key and user management:

```bash
# Enable OS Login at project level
gcloud compute project-info add-metadata \
  --metadata enable-oslogin=TRUE

# Enable OS Login for specific instance
gcloud compute instances add-metadata INSTANCE_NAME \
  --zone=ZONE \
  --metadata enable-oslogin=TRUE

# Grant OS Login roles
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:user@example.com \
  --role=roles/compute.osLogin

# SSH with OS Login
gcloud compute ssh INSTANCE_NAME --zone=ZONE
```

**Shielded VMs:**
Advanced security features for VM instances:

```bash
# Create Shielded VM
gcloud compute instances create secure-vm \
  --zone=us-central1-a \
  --machine-type=n1-standard-1 \
  --image-family=ubuntu-2004-lts \
  --image-project=ubuntu-os-cloud \
  --shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring

# Update Shielded VM policy
gcloud compute instances update secure-vm \
  --zone=us-central1-a \
  --shielded-learn-integrity-policy
```

**Confidential Computing:**
Memory encryption for sensitive workloads:

```bash
# Create Confidential VM
gcloud compute instances create confidential-vm \
  --zone=us-central1-a \
  --machine-type=n2d-standard-2 \
  --confidential-compute \
  --maintenance-policy=TERMINATE \
  --image-family=ubuntu-2004-lts \
  --image-project=ubuntu-os-cloud
```

### Container Security

**Container Image Security:**
Secure container image practices:

**Binary Authorization:**
Ensure only trusted container images are deployed:

```bash
# Create attestor
gcloud container binauthz attestors create prod-attestor \
  --attestation-authority-note=projects/PROJECT_ID/notes/prod-note \
  --description="Production attestor"

# Create policy
gcloud container binauthz policy import policy.yaml

# policy.yaml
defaultAdmissionRule:
  requireAttestationsBy:
  - projects/PROJECT_ID/attestors/prod-attestor
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
clusterAdmissionRules:
  us-central1-a.prod-cluster:
    requireAttestationsBy:
    - projects/PROJECT_ID/attestors/prod-attestor
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
```

**Container Analysis API:**
Vulnerability scanning for container images:

```bash
# Enable Container Analysis API
gcloud services enable containeranalysis.googleapis.com

# Scan image
gcloud container images scan IMAGE_URL

# Get scan results
gcloud container images list-tags IMAGE_URL \
  --format="get(digest)" \
  --limit=1 | xargs -I {} gcloud container images describe IMAGE_URL@{}
```

### App Engine Security

**App Engine Firewall:**
Application-level firewall rules:

```bash
# Create firewall rule
gcloud app firewall-rules create 1000 \
  --source-range=203.0.113.0/24 \
  --action=allow \
  --description="Allow office IP"

# Deny all other traffic
gcloud app firewall-rules create 2000 \
  --source-range=* \
  --action=deny \
  --description="Deny all other traffic"
```

**Identity-Aware Proxy (IAP):**
Application-level access control:

```bash
# Enable IAP
gcloud iap web enable \
  --resource-type=app-engine

# Grant IAP access
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:user@example.com \
  --role=roles/iap.httpsResourceAccessor
```

---

## 🔹 Container & Kubernetes Security

### GKE Security Features

**GKE Autopilot:**
Fully managed, secure-by-default Kubernetes:

```bash
# Create Autopilot cluster
gcloud container clusters create-auto secure-autopilot \
  --region=us-central1 \
  --enable-network-policy \
  --enable-ip-alias \
  --enable-autorepair \
  --enable-autoupgrade
```

**Private GKE Clusters:**
Network isolation for Kubernetes clusters:

```bash
# Create private cluster
gcloud container clusters create secure-private-cluster \
  --zone=us-central1-a \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-ip-alias \
  --enable-network-policy \
  --enable-autorepair \
  --enable-autoupgrade \
  --enable-shielded-nodes
```

**Workload Identity:**
Secure access to Google Cloud services from GKE:

```bash
# Enable Workload Identity on cluster
gcloud container clusters update CLUSTER_NAME \
  --zone=ZONE \
  --workload-pool=PROJECT_ID.svc.id.goog

# Create Kubernetes service account
kubectl create serviceaccount KSA_NAME

# Create Google service account
gcloud iam service-accounts create GSA_NAME

# Bind service accounts
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]" \
  GSA_NAME@PROJECT_ID.iam.gserviceaccount.com

# Annotate Kubernetes service account
kubectl annotate serviceaccount KSA_NAME \
  iam.gke.io/gcp-service-account=GSA_NAME@PROJECT_ID.iam.gserviceaccount.com
```

### Pod Security Standards

**Pod Security Policy (Deprecated) to Pod Security Standards:**

**Pod Security Standards Levels:**
- **Privileged**: Unrestricted policy
- **Baseline**: Minimally restrictive policy
- **Restricted**: Heavily restricted policy

**Implementation:**
```yaml
# Namespace with Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Secure Pod Configuration:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: secure-namespace
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.20
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "128Mi"
        cpu: "100m"
      requests:
        memory: "64Mi"
        cpu: "50m"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

### Network Policies

**Kubernetes Network Policies:**
Micro-segmentation for pod-to-pod communication:

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress

# Allow specific communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

# Allow egress to specific services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

### Service Mesh Security

**Istio Security Features:**
Comprehensive service mesh security:

**Mutual TLS (mTLS):**
```yaml
# Enable mTLS for entire mesh
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

# Service-specific mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: backend-mtls
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  mtls:
    mode: STRICT
```

**Authorization Policies:**
```yaml
# Deny all by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec: {}

# Allow specific access
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend"]
  - to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

---

## 🔹 Application Security & DevSecOps

### Secure Development Practices

**Security Scanning in CI/CD:**
Integrate security scanning into development pipeline:

**Cloud Build Security Pipeline:**
```yaml
# cloudbuild.yaml
steps:
# Build application
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA', '.']

# Security scan with Trivy
- name: 'aquasec/trivy'
  args: ['image', '--exit-code', '1', '--severity', 'HIGH,CRITICAL', 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA']

# SAST scan with SonarQube
- name: 'gcr.io/$PROJECT_ID/sonarqube-scanner'
  env:
  - 'SONAR_HOST_URL=https://sonarqube.example.com'
  - 'SONAR_LOGIN=$_SONAR_TOKEN'

# Push image if scans pass
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/app:$COMMIT_SHA']

# Deploy to staging
- name: 'gcr.io/cloud-builders/gke-deploy'
  args:
  - run
  - --filename=k8s/
  - --image=gcr.io/$PROJECT_ID/app:$COMMIT_SHA
  - --cluster=staging-cluster
  - --location=us-central1-a

substitutions:
  _SONAR_TOKEN: 'projects/$PROJECT_ID/secrets/sonar-token/versions/latest'

options:
  logging: CLOUD_LOGGING_ONLY
```

**Container Registry Vulnerability Scanning:**
```bash
# Enable vulnerability scanning
gcloud container images scan IMAGE_URL

# Get vulnerability report
gcloud container images describe IMAGE_URL \
  --show-package-vulnerability
```

### Web Application Security

**Cloud Armor:**
DDoS protection and WAF for applications:

```bash
# Create security policy
gcloud compute security-policies create web-app-policy \
  --description="Web application security policy"

# Add OWASP Top 10 protection
gcloud compute security-policies rules create 1000 \
  --security-policy=web-app-policy \
  --expression="evaluatePreconfiguredExpr('sqli-stable')" \
  --action=deny-403 \
  --description="SQL injection protection"

gcloud compute security-policies rules create 1001 \
  --security-policy=web-app-policy \
  --expression="evaluatePreconfiguredExpr('xss-stable')" \
  --action=deny-403 \
  --description="XSS protection"

# Rate limiting
gcloud compute security-policies rules create 2000 \
  --security-policy=web-app-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600 \
  --conform-action=allow \
  --exceed-action=deny-429

# Geographic restrictions
gcloud compute security-policies rules create 3000 \
  --security-policy=web-app-policy \
  --expression="origin.region_code == 'CN'" \
  --action=deny-403 \
  --description="Block traffic from China"
```

**reCAPTCHA Enterprise:**
Advanced bot protection:

```bash
# Create reCAPTCHA key
gcloud recaptcha keys create \
  --display-name="Web App Key" \
  --web \
  --domains=example.com,www.example.com \
  --integration-type=SCORE \
  --waf-feature=CHALLENGE_PAGE
```

### API Security

**Cloud Endpoints:**
API management and security:

```yaml
# openapi.yaml
swagger: '2.0'
info:
  title: Secure API
  version: 1.0.0
host: api.example.com
schemes:
  - https
security:
  - api_key: []
securityDefinitions:
  api_key:
    type: apiKey
    name: key
    in: query
paths:
  /users:
    get:
      operationId: getUsers
      security:
        - api_key: []
      responses:
        200:
          description: Success
```

```bash
# Deploy API configuration
gcloud endpoints services deploy openapi.yaml

# Create API key
gcloud alpha services api-keys create \
  --display-name="API Key" \
  --api-restrictions-service=api.example.com
```

**API Gateway:**
Fully managed API gateway:

```yaml
# api-config.yaml
swagger: '2.0'
info:
  title: Gateway API
  version: 1.0.0
x-google-backend:
  address: https://backend.example.com
security:
  - google_id_token: []
securityDefinitions:
  google_id_token:
    authorizationUrl: ""
    flow: "implicit"
    type: "oauth2"
    x-google-issuer: "https://accounts.google.com"
    x-google-jwks_uri: "https://www.googleapis.com/oauth2/v3/certs"
    x-google-audiences: "CLIENT_ID"
```

---

## 🔹 Monitoring, Logging & Incident Response

### Security Monitoring

**Cloud Security Command Center (SCC):**
Centralized security management and monitoring:

```bash
# Enable Security Command Center API
gcloud services enable securitycenter.googleapis.com

# List findings
gcloud scc findings list ORGANIZATION_ID \
  --filter="state=\"ACTIVE\" AND category=\"MALWARE\""

# Create custom finding
gcloud scc findings create FINDING_ID \
  --organization=ORGANIZATION_ID \
  --source=SOURCE_ID \
  --category="CUSTOM_SECURITY_ISSUE" \
  --state=ACTIVE \
  --resource-name="//compute.googleapis.com/projects/PROJECT_ID/zones/us-central1-a/instances/INSTANCE_NAME"
```

**Cloud Logging Security:**
Comprehensive audit logging:

```bash
# Create log sink for security events
gcloud logging sinks create security-sink \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/security_logs \
  --log-filter='protoPayload.serviceName="compute.googleapis.com" OR protoPayload.serviceName="iam.googleapis.com"'

# Create alerting policy
gcloud alpha monitoring policies create \
  --policy-from-file=security-alert-policy.yaml
```

```yaml
# security-alert-policy.yaml
displayName: "Failed Login Attempts"
conditions:
  - displayName: "High failed login rate"
    conditionThreshold:
      filter: 'resource.type="gce_instance" AND jsonPayload.event="login_failure"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 10
      duration: 300s
notificationChannels:
  - "projects/PROJECT_ID/notificationChannels/CHANNEL_ID"
```

### Threat Detection

**Event Threat Detection:**
Automated threat detection for Google Cloud:

**Detection Categories:**
- **Malware**: Malicious software detection
- **Cryptomining**: Unauthorized cryptocurrency mining
- **DDoS**: Distributed denial of service attacks
- **Brute Force**: Password brute force attacks
- **Data Exfiltration**: Unusual data access patterns

**Custom Detection Rules:**
```yaml
# Custom detection rule
rule:
  name: "Suspicious API Activity"
  description: "Detect unusual API call patterns"
  severity: "HIGH"
  condition: |
    protoPayload.serviceName="compute.googleapis.com" AND
    protoPayload.methodName="compute.instances.delete" AND
    protoPayload.authenticationInfo.principalEmail!~".*@example.com$"
  notification:
    - type: "email"
      destination: "security@example.com"
    - type: "pubsub"
      topic: "projects/PROJECT_ID/topics/security-alerts"
```

### Incident Response

**Automated Incident Response:**
Cloud Functions for automated response:

```python
# Cloud Function for incident response
import json
from google.cloud import compute_v1
from google.cloud import logging

def incident_response(event, context):
    """Automated incident response function"""
    
    # Parse the alert
    alert_data = json.loads(event['data'])
    
    # Initialize clients
    compute_client = compute_v1.InstancesClient()
    logging_client = logging.Client()
    
    if alert_data['finding']['category'] == 'MALWARE':
        # Isolate infected instance
        instance_name = extract_instance_name(alert_data)
        project_id = alert_data['finding']['resourceName'].split('/')[1]
        zone = extract_zone(alert_data)
        
        # Stop the instance
        operation = compute_client.stop(
            project=project_id,
            zone=zone,
            instance=instance_name
        )
        
        # Create forensic snapshot
        disk_name = f"{instance_name}-boot"
        snapshot_name = f"forensic-{instance_name}-{int(time.time())}"
        
        disks_client = compute_v1.DisksClient()
        snapshot_operation = disks_client.create_snapshot(
            project=project_id,
            zone=zone,
            disk=disk_name,
            snapshot_resource={'name': snapshot_name}
        )
        
        # Log the response action
        logger = logging_client.logger('incident-response')
        logger.info(f"Isolated instance {instance_name} due to malware detection")
        
        # Send notification
        send_notification(f"Instance {instance_name} isolated due to malware")

def extract_instance_name(alert_data):
    resource_name = alert_data['finding']['resourceName']
    return resource_name.split('/')[-1]

def extract_zone(alert_data):
    resource_name = alert_data['finding']['resourceName']
    return resource_name.split('/')[5]

def send_notification(message):
    # Send to Slack, email, or other notification system
    pass
```

**Forensic Analysis:**
```bash
# Create forensic disk from snapshot
gcloud compute disks create forensic-disk \
  --source-snapshot=forensic-snapshot \
  --zone=us-central1-a

# Attach to forensic analysis VM
gcloud compute instances attach-disk forensic-vm \
  --disk=forensic-disk \
  --zone=us-central1-a \
  --mode=ro
```

---

## 🔹 Compliance & Governance

### Compliance Frameworks

**SOC 2 Type II Compliance:**
Security, availability, and confidentiality controls:

**Control Implementation:**
- **Access Controls**: IAM policies and MFA
- **System Operations**: Monitoring and alerting
- **Change Management**: Infrastructure as Code
- **Risk Management**: Regular security assessments

**HIPAA Compliance:**
Healthcare data protection requirements:

**Technical Safeguards:**
```bash
# Enable audit logging for HIPAA
gcloud logging sinks create hipaa-audit-sink \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/hipaa_audit \
  --log-filter='protoPayload.serviceName="healthcare.googleapis.com" OR protoPayload.serviceName="storage.googleapis.com"'

# Create DLP job for PHI detection
gcloud dlp jobs create inspect \
  --location=global \
  --job-config=hipaa-dlp-config.json
```

```json
{
  "inspectConfig": {
    "infoTypes": [
      {"name": "US_HEALTHCARE_NPI"},
      {"name": "US_SOCIAL_SECURITY_NUMBER"},
      {"name": "DATE_OF_BIRTH"}
    ],
    "minLikelihood": "LIKELY",
    "includeQuote": true
  },
  "storageConfig": {
    "cloudStorageOptions": {
      "fileSet": {
        "url": "gs://healthcare-data-bucket/*"
      }
    }
  }
}
```

**PCI DSS Compliance:**
Payment card industry data security:

**Network Segmentation:**
```bash
# Create isolated network for PCI workloads
gcloud compute networks create pci-network --subnet-mode=custom

gcloud compute networks subnets create pci-subnet \
  --network=pci-network \
  --range=10.10.0.0/24 \
  --region=us-central1

# Restrictive firewall rules
gcloud compute firewall-rules create pci-deny-all \
  --network=pci-network \
  --action=deny \
  --rules=all \
  --source-ranges=0.0.0.0/0 \
  --priority=1000

gcloud compute firewall-rules create pci-allow-https \
  --network=pci-network \
  --action=allow \
  --rules=tcp:443 \
  --source-ranges=203.0.113.0/24 \
  --target-tags=pci-web \
  --priority=100
```

### Data Loss Prevention (DLP)

**Cloud DLP:**
Discover, classify, and protect sensitive data:

**DLP Templates:**
```bash
# Create inspection template
gcloud dlp inspect-templates create \
  --location=global \
  --inspect-template-config=dlp-template.json

# Create de-identification template
gcloud dlp deidentify-templates create \
  --location=global \
  --deidentify-template-config=deidentify-template.json
```

```json
{
  "displayName": "PII Detection Template",
  "description": "Template for detecting PII in data",
  "inspectConfig": {
    "infoTypes": [
      {"name": "EMAIL_ADDRESS"},
      {"name": "PHONE_NUMBER"},
      {"name": "CREDIT_CARD_NUMBER"},
      {"name": "US_SOCIAL_SECURITY_NUMBER"}
    ],
    "minLikelihood": "POSSIBLE",
    "limits": {
      "maxFindingsPerRequest": 100
    },
    "includeQuote": true
  }
}
```

**Automated DLP Scanning:**
```python
# Cloud Function for automated DLP scanning
from google.cloud import dlp_v2
from google.cloud import storage

def dlp_scan_new_files(event, context):
    """Scan new files uploaded to Cloud Storage"""
    
    dlp_client = dlp_v2.DlpServiceClient()
    storage_client = storage.Client()
    
    bucket_name = event['bucket']
    file_name = event['name']
    
    # Configure DLP inspection
    inspect_config = {
        "info_types": [
            {"name": "EMAIL_ADDRESS"},
            {"name": "PHONE_NUMBER"},
            {"name": "CREDIT_CARD_NUMBER"}
        ],
        "min_likelihood": dlp_v2.Likelihood.POSSIBLE,
        "include_quote": True,
    }
    
    # Configure storage
    storage_config = {
        "cloud_storage_options": {
            "file_set": {"url": f"gs://{bucket_name}/{file_name}"}
        }
    }
    
    # Create DLP job
    parent = f"projects/{PROJECT_ID}/locations/global"
    operation = dlp_client.create_dlp_job(
        request={
            "parent": parent,
            "inspect_job": {
                "inspect_config": inspect_config,
                "storage_config": storage_config,
            },
        }
    )
    
    print(f"DLP job created: {operation.name}")
```

### Resource Hierarchy and Organization

**Organization Policies:**
Centralized policy management:

```bash
# List available constraints
gcloud resource-manager org-policies list --organization=ORGANIZATION_ID

# Set VM external IP policy
gcloud resource-manager org-policies set-policy \
  --organization=ORGANIZATION_ID \
  vm-external-ip-policy.yaml
```

```yaml
# vm-external-ip-policy.yaml
constraint: constraints/compute.vmExternalIpAccess
booleanPolicy:
  enforced: true
```

**Resource Hierarchy Best Practices:**
```
Organization (company.com)
├── Security Folder
│   ├── Security Project (logging, monitoring)
│   └── Compliance Project (DLP, audit)
├── Production Folder
│   ├── Web App Project
│   └── Database Project
├── Staging Folder
│   ├── Staging Web Project
│   └── Staging DB Project
└── Development Folder
    ├── Dev Project 1
    └── Dev Project 2
```

---

## 🔹 Security Automation & Infrastructure as Code

### Terraform Security Modules

**Secure VPC Module:**
```hcl
# modules/secure-vpc/main.tf
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "network_name" {
  description = "VPC network name"
  type        = string
}

variable "subnets" {
  description = "Subnet configurations"
  type = map(object({
    ip_cidr_range = string
    region        = string
    secondary_ranges = map(string)
  }))
}

# Create VPC network
resource "google_compute_network" "vpc" {
  name                    = var.network_name
  auto_create_subnetworks = false
  project                 = var.project_id
}

# Create subnets
resource "google_compute_subnetwork" "subnets" {
  for_each = var.subnets
  
  name          = each.key
  ip_cidr_range = each.value.ip_cidr_range
  region        = each.value.region
  network       = google_compute_network.vpc.id
  project       = var.project_id
  
  # Enable private Google access
  private_ip_google_access = true
  
  # Enable flow logs
  log_config {
    aggregation_interval = "INTERVAL_10_MIN"
    flow_sampling        = 0.5
    metadata            = "INCLUDE_ALL_METADATA"
  }
  
  # Secondary ranges for GKE
  dynamic "secondary_ip_range" {
    for_each = each.value.secondary_ranges
    content {
      range_name    = secondary_ip_range.key
      ip_cidr_range = secondary_ip_range.value
    }
  }
}

# Default firewall rules
resource "google_compute_firewall" "deny_all_ingress" {
  name    = "${var.network_name}-deny-all-ingress"
  network = google_compute_network.vpc.name
  project = var.project_id
  
  deny {
    protocol = "all"
  }
  
  source_ranges = ["0.0.0.0/0"]
  priority      = 65534
}

resource "google_compute_firewall" "allow_internal" {
  name    = "${var.network_name}-allow-internal"
  network = google_compute_network.vpc.name
  project = var.project_id
  
  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
  
  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }
  
  allow {
    protocol = "icmp"
  }
  
  source_ranges = [
    for subnet in var.subnets : subnet.ip_cidr_range
  ]
  
  priority = 1000
}

output "network_id" {
  value = google_compute_network.vpc.id
}

output "subnet_ids" {
  value = {
    for k, v in google_compute_subnetwork.subnets : k => v.id
  }
}
```

**Secure GKE Module:**
```hcl
# modules/secure-gke/main.tf
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "cluster_name" {
  description = "GKE cluster name"
  type        = string
}

variable "network" {
  description = "VPC network"
  type        = string
}

variable "subnetwork" {
  description = "VPC subnetwork"
  type        = string
}

variable "master_ipv4_cidr_block" {
  description = "Master IPv4 CIDR block"
  type        = string
  default     = "172.16.0.0/28"
}

resource "google_container_cluster" "secure_cluster" {
  name     = var.cluster_name
  location = var.region
  project  = var.project_id
  
  # Remove default node pool
  remove_default_node_pool = true
  initial_node_count       = 1
  
  # Network configuration
  network    = var.network
  subnetwork = var.subnetwork
  
  # Private cluster configuration
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = var.master_ipv4_cidr_block
  }
  
  # IP allocation policy
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }
  
  # Network policy
  network_policy {
    enabled = true
  }
  
  # Enable Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  # Enable Shielded Nodes
  enable_shielded_nodes = true
  
  # Master auth configuration
  master_auth {
    client_certificate_config {
      issue_client_certificate = false
    }
  }
  
  # Addons configuration
  addons_config {
    http_load_balancing {
      disabled = false
    }
    
    horizontal_pod_autoscaling {
      disabled = false
    }
    
    network_policy_config {
      disabled = false
    }
    
    istio_config {
      disabled = false
      auth     = "AUTH_MUTUAL_TLS"
    }
  }
  
  # Binary authorization
  enable_binary_authorization = true
  
  # Database encryption
  database_encryption {
    state    = "ENCRYPTED"
    key_name = var.kms_key_id
  }
  
  # Maintenance policy
  maintenance_policy {
    recurring_window {
      start_time = "2023-01-01T02:00:00Z"
      end_time   = "2023-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"
    }
  }
}

resource "google_container_node_pool" "secure_nodes" {
  name       = "${var.cluster_name}-secure-pool"
  location   = var.region
  cluster    = google_container_cluster.secure_cluster.name
  project    = var.project_id
  node_count = 1
  
  # Auto-scaling
  autoscaling {
    min_node_count = 1
    max_node_count = 10
  }
  
  # Node configuration
  node_config {
    preemptible  = false
    machine_type = "e2-medium"
    
    # Service account
    service_account = google_service_account.gke_nodes.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring",
    ]
    
    # Shielded instance config
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
    
    # Workload metadata config
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
    
    # Labels and tags
    labels = {
      environment = "production"
      managed-by  = "terraform"
    }
    
    tags = ["gke-node", "secure"]
  }
  
  # Management
  management {
    auto_repair  = true
    auto_upgrade = true
  }
  
  # Upgrade settings
  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
  }
}

resource "google_service_account" "gke_nodes" {
  account_id   = "${var.cluster_name}-nodes"
  display_name = "GKE Nodes Service Account"
  project      = var.project_id
}

resource "google_project_iam_member" "gke_nodes_roles" {
  for_each = toset([
    "roles/logging.logWriter",
    "roles/monitoring.metricWriter",
    "roles/monitoring.viewer",
    "roles/stackdriver.resourceMetadata.writer"
  ])
  
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}
```

### Security Policy as Code

**Organization Policy Module:**
```hcl
# modules/org-policies/main.tf
variable "organization_id" {
  description = "Organization ID"
  type        = string
}

variable "folder_id" {
  description = "Folder ID (optional)"
  type        = string
  default     = null
}

variable "project_id" {
  description = "Project ID (optional)"
  type        = string
  default     = null
}

locals {
  parent = var.folder_id != null ? "folders/${var.folder_id}" : var.project_id != null ? "projects/${var.project_id}" : "organizations/${var.organization_id}"
}

# Restrict VM external IP access
resource "google_org_policy_policy" "vm_external_ip_access" {
  name   = "${local.parent}/policies/compute.vmExternalIpAccess"
  parent = local.parent
  
  spec {
    rules {
      enforce = "TRUE"
    }
  }
}

# Require OS Login
resource "google_org_policy_policy" "require_os_login" {
  name   = "${local.parent}/policies/compute.requireOsLogin"
  parent = local.parent
  
  spec {
    rules {
      enforce = "TRUE"
    }
  }
}

# Restrict allowed machine types
resource "google_org_policy_policy" "allowed_machine_types" {
  name   = "${local.parent}/policies/compute.restrictMachineTypes"
  parent = local.parent
  
  spec {
    rules {
      values {
        allowed_values = [
          "projects/*/zones/*/machineTypes/e2-micro",
          "projects/*/zones/*/machineTypes/e2-small",
          "projects/*/zones/*/machineTypes/e2-medium"
        ]
      }
    }
  }
}

# Disable service account key creation
resource "google_org_policy_policy" "disable_sa_key_creation" {
  name   = "${local.parent}/policies/iam.disableServiceAccountKeyCreation"
  parent = local.parent
  
  spec {
    rules {
      enforce = "TRUE"
    }
  }
}

# Restrict resource locations
resource "google_org_policy_policy" "resource_locations" {
  name   = "${local.parent}/policies/gcp.resourceLocations"
  parent = local.parent
  
  spec {
    rules {
      values {
        allowed_values = [
          "in:us-locations",
          "in:eu-locations"
        ]
      }
    }
  }
}
```

This comprehensive GCP Cloud Security guide provides enterprise-level security concepts and practices essential for senior DevOps engineers managing secure cloud infrastructure. The focus is on deep understanding of security architecture, compliance requirements, and integration with modern DevSecOps practices.

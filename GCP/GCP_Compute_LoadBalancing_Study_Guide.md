# GCP Compute Engine & Load Balancing Study Guide
*Comprehensive guide for Senior DevOps Engineers*

## Table of Contents
1. [Google Compute Engine (VM Basics)](#google-compute-engine-vm-basics)
2. [Instance Groups & Autoscaling](#instance-groups--autoscaling)
3. [Load Balancing in GCP](#load-balancing-in-gcp)
4. [High Availability & Scalability](#high-availability--scalability)
5. [Cost Optimization & Billing](#cost-optimization--billing)
6. [Security & Best Practices](#security--best-practices)
7. [Advanced Topics](#advanced-topics)

---

## Google Compute Engine (VM Basics)

### What is Google Compute Engine (GCE)?
Google Compute Engine is GCP's Infrastructure-as-a-Service (IaaS) offering that provides scalable, high-performance virtual machines running in Google's data centers.

**Why use GCE?**
- **Scalability**: Scale from single instances to thousands
- **Performance**: Custom machine types, GPUs, high-performance networking
- **Cost-effective**: Per-second billing, sustained use discounts
- **Integration**: Native integration with other GCP services
- **Global**: Available in multiple regions and zones

**Example Use Case**: E-commerce platform needing web servers, database servers, and background processing workers.

### Creating Virtual Machines

#### Console Method:
1. Navigate to Compute Engine → VM instances
2. Click "Create Instance"
3. Configure:
   - Name: `web-server-01`
   - Region/Zone: `us-central1-a`
   - Machine type: `e2-medium`
   - Boot disk: `Ubuntu 20.04 LTS`
   - Firewall: Allow HTTP/HTTPS traffic

#### gcloud Command:
```bash
# Create a basic VM
gcloud compute instances create web-server-01 \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --tags=http-server,https-server

# Create VM with startup script
gcloud compute instances create web-server-02 \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --metadata-from-file startup-script=startup.sh \
    --tags=http-server
```

### Machine Types and Images

#### Machine Types:
- **General Purpose**: `e2`, `n1`, `n2`, `n2d` - Balanced CPU/memory
- **Compute Optimized**: `c2` - High-performance computing
- **Memory Optimized**: `m1`, `m2` - In-memory databases, analytics
- **Accelerator Optimized**: `a2` - ML workloads with GPUs

**Examples**:
```bash
# Standard machine type
--machine-type=e2-standard-4  # 4 vCPUs, 16GB RAM

# Custom machine type
--custom-cpu=6 --custom-memory=20GB

# High-memory machine
--machine-type=n1-highmem-8  # 8 vCPUs, 52GB RAM
```

#### Images:
- **Public Images**: Ubuntu, CentOS, Windows Server
- **Custom Images**: Your own configured images
- **Container Images**: Container-Optimized OS

### Installing HTTP Web Server

```bash
# Connect to VM
gcloud compute ssh web-server-01 --zone=us-central1-a

# Install Apache
sudo apt update
sudo apt install apache2 -y
sudo systemctl start apache2
sudo systemctl enable apache2

# Create simple webpage
echo "<h1>Hello from $(hostname)</h1>" | sudo tee /var/www/html/index.html

# Configure firewall
sudo ufw allow 'Apache Full'
```

### IP Addresses

#### Internal vs External IPs:

**Internal IP**:
- Private IP within VPC network
- Used for communication between GCP resources
- Example: `10.128.0.2`
- Free, automatically assigned

**External IP**:
- Public IP accessible from internet
- Can be ephemeral (changes on restart) or static
- Example: `34.123.45.67`
- Charged when not attached to running instance

#### Static IP Address:
```bash
# Reserve static IP
gcloud compute addresses create web-static-ip --region=us-central1

# Assign to VM
gcloud compute instances add-access-config web-server-01 \
    --zone=us-central1-a \
    --access-config-name="External NAT" \
    --address=web-static-ip
```

**When to use Static IP**:
- DNS records pointing to your service
- SSL certificates tied to specific IP
- Firewall rules based on IP addresses
- Load balancer frontend IPs

### Startup Scripts

Startup scripts automate VM configuration during boot.

**Example startup script** (`startup.sh`):
```bash
#!/bin/bash
# Update system
apt-get update

# Install Apache
apt-get install -y apache2

# Install PHP
apt-get install -y php libapache2-mod-php

# Create application directory
mkdir -p /var/www/html/app

# Download application code
gsutil cp gs://my-app-bucket/app.tar.gz /tmp/
tar -xzf /tmp/app.tar.gz -C /var/www/html/app/

# Set permissions
chown -R www-data:www-data /var/www/html/app/
chmod -R 755 /var/www/html/app/

# Restart Apache
systemctl restart apache2
systemctl enable apache2

# Log completion
echo "Startup script completed at $(date)" >> /var/log/startup.log
```

### Instance Templates

Instance templates define VM configuration for consistent deployments.

```bash
# Create instance template
gcloud compute instance-templates create web-template \
    --machine-type=e2-medium \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --metadata-from-file startup-script=startup.sh \
    --tags=http-server \
    --scopes=storage-ro

# Create VM from template
gcloud compute instances create web-server-03 \
    --source-instance-template=web-template \
    --zone=us-central1-a
```

### Custom Images

Custom images reduce launch time and ensure consistency.

```bash
# Create image from existing VM
gcloud compute images create web-server-image \
    --source-disk=web-server-01 \
    --source-disk-zone=us-central1-a \
    --description="Pre-configured web server with Apache and PHP"

# Create VM from custom image
gcloud compute instances create web-server-04 \
    --image=web-server-image \
    --zone=us-central1-a
```

### Troubleshooting VM Issues

**Common startup issues**:

1. **Apache not running**:
```bash
# Check service status
sudo systemctl status apache2

# Check logs
sudo journalctl -u apache2

# Check ports
sudo netstat -tlnp | grep :80
```

2. **Startup script failures**:
```bash
# Check startup script logs
sudo journalctl -u google-startup-scripts

# Check metadata
curl "http://metadata.google.internal/computeMetadata/v1/instance/attributes/startup-script" -H "Metadata-Flavor: Google"
```

3. **Network connectivity**:
```bash
# Check firewall rules
gcloud compute firewall-rules list

# Test connectivity
curl -I http://EXTERNAL_IP
```

---

## Instance Groups & Autoscaling

### Managed Instance Groups (MIGs)

MIGs provide identical VMs that can be managed as a single entity.

**Benefits**:
- **Autoscaling**: Automatically add/remove instances
- **Auto-healing**: Replace unhealthy instances
- **Rolling updates**: Update instances without downtime
- **Load balancing**: Distribute traffic across instances

**Use Case Example**: Web application that needs to handle variable traffic loads.

### Creating Managed Instance Groups

```bash
# Create instance template first
gcloud compute instance-templates create web-app-template \
    --machine-type=e2-medium \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --metadata-from-file startup-script=web-startup.sh \
    --tags=http-server

# Create MIG
gcloud compute instance-groups managed create web-app-mig \
    --template=web-app-template \
    --size=3 \
    --zone=us-central1-a

# Set up autoscaling
gcloud compute instance-groups managed set-autoscaling web-app-mig \
    --zone=us-central1-a \
    --max-num-replicas=10 \
    --min-num-replicas=2 \
    --target-cpu-utilization=0.7
```

### Rolling Updates

Rolling updates allow you to update instances gradually without downtime.

```bash
# Create new template version
gcloud compute instance-templates create web-app-template-v2 \
    --machine-type=e2-standard-2 \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --metadata-from-file startup-script=web-startup-v2.sh

# Start rolling update
gcloud compute instance-groups managed rolling-action start-update web-app-mig \
    --zone=us-central1-a \
    --version=template=web-app-template-v2 \
    --max-surge=2 \
    --max-unavailable=1

# Monitor update progress
gcloud compute instance-groups managed describe web-app-mig --zone=us-central1-a
```

### Why Use Instance Groups vs Single VMs?

| Aspect | Single VMs | Instance Groups |
|--------|------------|-----------------|
| **Scalability** | Manual scaling | Automatic scaling |
| **Availability** | Single point of failure | Auto-healing, redundancy |
| **Management** | Individual management | Centralized management |
| **Updates** | Manual, risky | Rolling updates |
| **Load Distribution** | Manual setup | Integrated with load balancers |

---

## Load Balancing in GCP

### Introduction to Load Balancing

Load balancing distributes incoming traffic across multiple backend instances to ensure no single instance becomes overwhelmed.

**Why Load Balancing is Needed**:
- **High Availability**: Eliminate single points of failure
- **Scalability**: Handle increased traffic by adding backends
- **Performance**: Reduce response times through traffic distribution
- **Geographic Distribution**: Route users to nearest servers

### Role of Load Balancer

A load balancer acts as a reverse proxy that:
1. Receives client requests
2. Selects appropriate backend based on algorithm
3. Forwards request to selected backend
4. Returns response to client
5. Monitors backend health

### Types of Load Balancers in GCP

#### 1. HTTP(S) Load Balancer (Layer 7)
- **Global**: Single anycast IP, global distribution
- **Features**: URL-based routing, SSL termination, CDN integration
- **Use Cases**: Web applications, APIs, microservices

**Example Configuration**:
```bash
# Create health check
gcloud compute health-checks create http web-health-check \
    --port=80 \
    --request-path=/health

# Create backend service
gcloud compute backend-services create web-backend-service \
    --protocol=HTTP \
    --health-checks=web-health-check \
    --global

# Add instance group to backend
gcloud compute backend-services add-backend web-backend-service \
    --instance-group=web-app-mig \
    --instance-group-zone=us-central1-a \
    --global

# Create URL map
gcloud compute url-maps create web-url-map \
    --default-service=web-backend-service

# Create HTTP proxy
gcloud compute target-http-proxies create web-http-proxy \
    --url-map=web-url-map

# Create forwarding rule
gcloud compute forwarding-rules create web-forwarding-rule \
    --global \
    --target-http-proxy=web-http-proxy \
    --ports=80
```

#### 2. TCP/UDP Load Balancer (Layer 4)
- **Regional**: Operates within a region
- **Features**: Protocol-agnostic, preserves client IP
- **Use Cases**: Databases, gaming servers, custom protocols

**Example**:
```bash
# Create TCP health check
gcloud compute health-checks create tcp db-health-check \
    --port=3306

# Create backend service
gcloud compute backend-services create db-backend-service \
    --protocol=TCP \
    --health-checks=db-health-check \
    --region=us-central1

# Create forwarding rule
gcloud compute forwarding-rules create db-forwarding-rule \
    --region=us-central1 \
    --ports=3306 \
    --backend-service=db-backend-service
```

#### 3. Internal Load Balancer
- **Private**: Only accessible within VPC
- **Features**: Preserves source IP, low latency
- **Use Cases**: Internal microservices, database clusters

### Connecting VMs to Load Balancer

#### Step 1: Prepare Backend VMs (Inventory Service)
```bash
# Create inventory service template
gcloud compute instance-templates create inventory-template \
    --machine-type=e2-medium \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --metadata-from-file startup-script=inventory-startup.sh \
    --tags=inventory-service

# Create MIG for inventory service
gcloud compute instance-groups managed create inventory-mig \
    --template=inventory-template \
    --size=2 \
    --zone=us-central1-a
```

#### Step 2: Configure Load Balancer
```bash
# Create health check for inventory service
gcloud compute health-checks create http inventory-health-check \
    --port=8080 \
    --request-path=/api/health

# Create backend service
gcloud compute backend-services create inventory-backend \
    --protocol=HTTP \
    --health-checks=inventory-health-check \
    --global

# Add MIG to backend
gcloud compute backend-services add-backend inventory-backend \
    --instance-group=inventory-mig \
    --instance-group-zone=us-central1-a \
    --global
```

#### Step 3: Test Configuration
```bash
# Get load balancer IP
LB_IP=$(gcloud compute forwarding-rules describe inventory-lb --global --format="value(IPAddress)")

# Test connectivity
curl http://$LB_IP/api/inventory
```

### Blocking Direct Traffic to Backend VMs

**Why block direct access?**
- **Security**: Prevent bypassing load balancer security features
- **Monitoring**: Ensure all traffic goes through centralized monitoring
- **SSL Termination**: Force HTTPS at load balancer level

**Implementation**:
```bash
# Remove default allow rules for backends
gcloud compute firewall-rules delete default-allow-http

# Create restrictive firewall rule
gcloud compute firewall-rules create allow-lb-to-backends \
    --allow tcp:8080 \
    --source-ranges 130.211.0.0/22,35.191.0.0/16 \
    --target-tags inventory-service \
    --description "Allow only load balancer health checks and traffic"

# Block direct external access
gcloud compute firewall-rules create deny-direct-access \
    --action DENY \
    --rules tcp:8080 \
    --source-ranges 0.0.0.0/0 \
    --target-tags inventory-service \
    --priority 1000
```

### Adding Multiple Services (Catalog Service)

```bash
# Create catalog service template
gcloud compute instance-templates create catalog-template \
    --machine-type=e2-medium \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --metadata-from-file startup-script=catalog-startup.sh \
    --tags=catalog-service

# Create catalog MIG
gcloud compute instance-groups managed create catalog-mig \
    --template=catalog-template \
    --size=2 \
    --zone=us-central1-b

# Create catalog backend service
gcloud compute backend-services create catalog-backend \
    --protocol=HTTP \
    --health-checks=catalog-health-check \
    --global

# Update URL map for path-based routing
gcloud compute url-maps edit main-url-map
```

**URL Map Configuration**:
```yaml
defaultService: projects/PROJECT_ID/global/backendServices/inventory-backend
pathMatchers:
- name: main-matcher
  defaultService: projects/PROJECT_ID/global/backendServices/inventory-backend
  pathRules:
  - paths: ['/api/catalog/*']
    service: projects/PROJECT_ID/global/backendServices/catalog-backend
  - paths: ['/api/inventory/*']
    service: projects/PROJECT_ID/global/backendServices/inventory-backend
```

### Load Balancer Terminologies

- **Backend Service**: Defines how load balancer distributes traffic
- **Health Check**: Monitors backend instance health
- **Forwarding Rule**: Specifies IP, port, and target proxy
- **URL Map**: Defines routing rules (HTTP/HTTPS only)
- **Target Proxy**: Terminates client connections
- **Instance Group**: Collection of VM instances serving as backends

### Multi-Region Load Balancing

```bash
# Create instance groups in multiple regions
gcloud compute instance-groups managed create web-mig-us \
    --template=web-template \
    --size=2 \
    --zone=us-central1-a

gcloud compute instance-groups managed create web-mig-eu \
    --template=web-template \
    --size=2 \
    --zone=europe-west1-b

# Add both regions to backend service
gcloud compute backend-services add-backend web-backend \
    --instance-group=web-mig-us \
    --instance-group-zone=us-central1-a \
    --global

gcloud compute backend-services add-backend web-backend \
    --instance-group=web-mig-eu \
    --instance-group-zone=europe-west1-b \
    --global
```

### Session Affinity

Session affinity ensures requests from the same client go to the same backend.

**Types**:
- **None**: No affinity (default)
- **Client IP**: Based on client IP address
- **Generated Cookie**: Load balancer generates cookie
- **Client IP and Protocol**: IP + protocol combination

```bash
# Configure session affinity
gcloud compute backend-services update web-backend \
    --session-affinity=CLIENT_IP \
    --global
```

**When to use**:
- Applications storing session data locally
- Stateful applications
- WebSocket connections

### Stateless Architecture Benefits

**Why stateless is recommended**:
- **Scalability**: Any instance can handle any request
- **Reliability**: No data loss when instances fail
- **Load Distribution**: Perfect load balancing possible
- **Deployment**: Easier rolling updates and scaling

**Implementation**:
- Store session data in external stores (Redis, Memcached)
- Use JWT tokens for authentication
- Design APIs to be stateless
- Use managed databases for persistent data

---

## High Availability & Scalability

### High Availability in GCP

High Availability (HA) ensures services remain operational despite failures.

**GCP HA Features**:
- **Multi-zone deployment**: Spread across availability zones
- **Live migration**: Move VMs during maintenance without downtime
- **Automatic restart**: Restart failed instances automatically
- **Health checks**: Monitor and replace unhealthy instances
- **Global load balancing**: Route around failed regions

### Implementing HA for Compute Engine

```bash
# Create regional MIG (spans multiple zones)
gcloud compute instance-groups managed create ha-web-mig \
    --template=web-template \
    --size=6 \
    --region=us-central1

# Configure distribution across zones
gcloud compute instance-groups managed set-target-pools ha-web-mig \
    --region=us-central1 \
    --target-pools=web-pool

# Set up auto-healing
gcloud compute instance-groups managed set-autohealing ha-web-mig \
    --health-check=web-health-check \
    --initial-delay=300 \
    --region=us-central1
```

### Vertical vs Horizontal Scaling

#### Vertical Scaling (Scale Up)
- **Definition**: Increase resources of existing instances
- **Pros**: Simple, no application changes needed
- **Cons**: Limited by machine type limits, single point of failure

```bash
# Resize existing VM
gcloud compute instances set-machine-type web-server-01 \
    --machine-type=n1-standard-4 \
    --zone=us-central1-a
```

#### Horizontal Scaling (Scale Out)
- **Definition**: Add more instances to handle load
- **Pros**: Unlimited scaling, better fault tolerance
- **Cons**: Requires stateless application design

```bash
# Configure autoscaling for horizontal scaling
gcloud compute instance-groups managed set-autoscaling web-mig \
    --max-num-replicas=20 \
    --min-num-replicas=3 \
    --target-cpu-utilization=0.6 \
    --zone=us-central1-a
```

### Live Migration and Automatic Restart

**Live Migration**:
- Moves running VMs to different physical hardware
- Zero downtime during maintenance
- Transparent to applications and users

**Automatic Restart**:
- Restarts VMs after host system failures
- Configurable per instance
- Works with persistent disks

```bash
# Configure maintenance policy
gcloud compute instances create resilient-vm \
    --maintenance-policy=MIGRATE \
    --restart-on-failure \
    --zone=us-central1-a
```

### GPUs in Compute Engine

**Use Cases**:
- Machine learning training and inference
- Scientific computing and simulations
- Video processing and rendering
- Cryptocurrency mining

```bash
# Create GPU-enabled VM
gcloud compute instances create ml-training-vm \
    --zone=us-central1-a \
    --machine-type=n1-standard-4 \
    --accelerator=type=nvidia-tesla-t4,count=1 \
    --image-family=tf-latest-gpu \
    --image-project=deeplearning-platform-release \
    --maintenance-policy=TERMINATE \
    --restart-on-failure
```

---

## Cost Optimization & Billing

### Sustained Use Discounts

Automatic discounts for running instances for significant portions of the month.

**How it works**:
- Automatic discount for instances running >25% of month
- Up to 30% discount for running entire month
- Applied to vCPUs and memory separately
- No upfront commitment required

**Example**:
```
Instance: n1-standard-4 (4 vCPUs, 15GB RAM)
List price: $0.1900/hour
Running 730 hours/month (100%): 30% discount = $0.1330/hour
Running 500 hours/month (68%): ~20% discount = $0.1520/hour
```

### Committed Use Discounts (CUDs)

Discounts for committing to use resources for 1 or 3 years.

**Types**:
- **Resource-based**: Commit to specific machine types in specific regions
- **Spend-based**: Commit to spend amount across Compute Engine

**Comparison**:
| Commitment | 1-Year Discount | 3-Year Discount |
|------------|----------------|-----------------|
| General Purpose | 25% | 52% |
| Memory Optimized | 25% | 52% |
| Compute Optimized | 25% | 52% |

```bash
# Purchase commitment
gcloud compute commitments create my-commitment \
    --plan=12-month \
    --region=us-central1 \
    --resources=type=VCPU,amount=100 \
    --resources=type=MEMORY,amount=400
```

### Preemptible (Spot) VMs

Short-lived instances at up to 80% discount.

**Characteristics**:
- Can be terminated with 30-second notice
- Maximum 24-hour runtime
- No SLA guarantees
- Perfect for fault-tolerant workloads

**Use Cases**:
- Batch processing jobs
- CI/CD pipelines
- Data analysis workloads
- Development and testing

```bash
# Create preemptible VM
gcloud compute instances create batch-worker \
    --preemptible \
    --zone=us-central1-a \
    --machine-type=n1-standard-4 \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud
```

### Compute Engine Billing Model

**Billing Components**:
1. **vCPUs**: Charged per vCPU per second
2. **Memory**: Charged per GB per second
3. **Persistent Disks**: Charged per GB per month
4. **Network**: Egress traffic charges
5. **External IPs**: Charged when not attached to running instance

**Example Calculation**:
```
n1-standard-4 in us-central1:
- 4 vCPUs × $0.031611/hour = $0.126444/hour
- 15 GB RAM × $0.004237/hour = $0.063555/hour
- Total: $0.189999/hour ≈ $0.19/hour

Monthly (730 hours): $138.70
With sustained use discount (30%): $97.09
```

### Cost Optimization Strategies

#### 1. Right-sizing
```bash
# Analyze VM utilization
gcloud compute instances list --format="table(name,zone,machineType,status)"

# Get recommendations
gcloud recommender recommendations list \
    --project=PROJECT_ID \
    --recommender=google.compute.instance.MachineTypeRecommender \
    --location=us-central1-a
```

#### 2. Scheduled Operations
```bash
# Create instance schedule
gcloud compute resource-policies create instance-schedule stop-nights \
    --region=us-central1 \
    --vm-stop-schedule="0 18 * * 1-5" \
    --vm-start-schedule="0 8 * * 1-5" \
    --timezone="America/New_York"

# Apply schedule to instances
gcloud compute instances add-resource-policies dev-server \
    --resource-policies=stop-nights \
    --zone=us-central1-a
```

#### 3. Storage Optimization
```bash
# Use balanced persistent disks (cheaper than SSD)
gcloud compute disks create app-disk \
    --size=100GB \
    --type=pd-balanced \
    --zone=us-central1-a

# Resize disks when needed
gcloud compute disks resize app-disk \
    --size=200GB \
    --zone=us-central1-a
```

#### 4. Network Cost Optimization
- Use internal IPs for inter-service communication
- Leverage Cloud CDN for static content
- Choose regions close to users
- Use regional persistent disks for multi-zone access

---

## Security & Best Practices

### Secure Network Design

#### 1. VPC and Firewall Configuration
```bash
# Create custom VPC
gcloud compute networks create secure-vpc \
    --subnet-mode=custom

# Create subnets with specific IP ranges
gcloud compute networks subnets create web-subnet \
    --network=secure-vpc \
    --range=10.1.1.0/24 \
    --region=us-central1

gcloud compute networks subnets create db-subnet \
    --network=secure-vpc \
    --range=10.1.2.0/24 \
    --region=us-central1

# Create restrictive firewall rules
gcloud compute firewall-rules create allow-web-traffic \
    --network=secure-vpc \
    --allow=tcp:80,tcp:443 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=web-server

gcloud compute firewall-rules create allow-db-from-web \
    --network=secure-vpc \
    --allow=tcp:3306 \
    --source-tags=web-server \
    --target-tags=db-server
```

#### 2. Identity and Access Management
```bash
# Create service account for VMs
gcloud iam service-accounts create vm-service-account \
    --display-name="VM Service Account"

# Grant minimal required permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:vm-service-account@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# Use service account in VM
gcloud compute instances create secure-vm \
    --service-account=vm-service-account@PROJECT_ID.iam.gserviceaccount.com \
    --scopes=storage-ro
```

#### 3. OS Security Hardening
```bash
# OS Login for centralized SSH key management
gcloud compute project-info add-metadata \
    --metadata enable-oslogin=TRUE

# Create VM with OS Login
gcloud compute instances create secure-vm \
    --metadata enable-oslogin=TRUE \
    --zone=us-central1-a
```

### Performance Best Practices

#### 1. Disk Performance
```bash
# Use SSD persistent disks for high IOPS
gcloud compute instances create high-perf-vm \
    --boot-disk-type=pd-ssd \
    --boot-disk-size=50GB \
    --zone=us-central1-a

# Attach additional SSD disk
gcloud compute disks create data-disk \
    --size=500GB \
    --type=pd-ssd \
    --zone=us-central1-a

gcloud compute instances attach-disk high-perf-vm \
    --disk=data-disk \
    --zone=us-central1-a
```

#### 2. Network Performance
```bash
# Enable IP forwarding for network appliances
gcloud compute instances create router-vm \
    --can-ip-forward \
    --zone=us-central1-a

# Use premium network tier for better performance
gcloud compute addresses create premium-ip \
    --network-tier=PREMIUM \
    --region=us-central1
```

#### 3. Load Balancer Optimization
```bash
# Configure connection draining
gcloud compute backend-services update web-backend \
    --connection-draining-timeout=300 \
    --global

# Set appropriate health check intervals
gcloud compute health-checks update http web-health-check \
    --check-interval=10s \
    --timeout=5s \
    --healthy-threshold=2 \
    --unhealthy-threshold=3
```

---

## Advanced Topics

### VM Manager

Automate OS patch management and configuration.

```bash
# Enable VM Manager API
gcloud services enable osconfig.googleapis.com

# Create patch deployment
gcloud compute os-config patch-deployments create monthly-patches \
    --file=patch-config.yaml

# Example patch-config.yaml
cat > patch-config.yaml << EOF
patchConfig:
  rebootConfig: REBOOT_IF_REQUIRED
  apt:
    type: DIST
instanceFilter:
  zones:
  - us-central1-a
  - us-central1-b
recurringSchedule:
  timeZone:
    id: America/New_York
  timeOfDay:
    hours: 2
  frequency: MONTHLY
  monthlySchedule:
    weekDayOfMonth:
      weekOrdinal: 2
      dayOfWeek: SUNDAY
EOF
```

### Sole Tenancy

Dedicated physical servers for compliance and licensing requirements.

```bash
# Create sole tenant node template
gcloud compute sole-tenancy node-templates create secure-template \
    --node-type=n1-node-96-624 \
    --region=us-central1

# Create sole tenant node group
gcloud compute sole-tenancy node-groups create secure-nodes \
    --node-template=secure-template \
    --target-size=2 \
    --zone=us-central1-a

# Create VM on sole tenant node
gcloud compute instances create secure-vm \
    --zone=us-central1-a \
    --node-group=secure-nodes
```

### Snapshots and Images

#### Automated Snapshot Schedule
```bash
# Create snapshot schedule
gcloud compute resource-policies create snapshot-schedule daily-backup \
    --region=us-central1 \
    --max-retention-days=7 \
    --on-source-disk-delete=KEEP_AUTO_SNAPSHOTS \
    --daily-schedule \
    --start-time=02:00 \
    --storage-location=us

# Apply to disk
gcloud compute disks add-resource-policies boot-disk \
    --resource-policies=daily-backup \
    --zone=us-central1-a
```

#### Custom Image Creation
```bash
# Prepare VM for imaging
gcloud compute instances stop source-vm --zone=us-central1-a

# Create custom image
gcloud compute images create golden-image-v1 \
    --source-disk=source-vm \
    --source-disk-zone=us-central1-a \
    --family=golden-images \
    --description="Golden image with security patches and applications"

# Share image across projects
gcloud compute images add-iam-policy-binding golden-image-v1 \
    --member='user:admin@company.com' \
    --role='roles/compute.imageUser'
```

### Monitoring and Logging

```bash
# Install monitoring agent
curl -sSO https://dl.google.com/cloudagents/add-monitoring-agent-repo.sh
sudo bash add-monitoring-agent-repo.sh
sudo apt-get update
sudo apt-get install stackdriver-agent

# Install logging agent
curl -sSO https://dl.google.com/cloudagents/add-logging-agent-repo.sh
sudo bash add-logging-agent-repo.sh
sudo apt-get update
sudo apt-get install google-fluentd
```

### Disaster Recovery

#### Multi-Region Backup Strategy
```bash
# Create cross-region snapshot
gcloud compute snapshots create dr-snapshot-$(date +%Y%m%d) \
    --source-disk=production-disk \
    --source-disk-zone=us-central1-a \
    --storage-location=us-east1

# Create disk from snapshot in different region
gcloud compute disks create dr-disk \
    --source-snapshot=dr-snapshot-20231201 \
    --zone=us-east1-b
```

---

## Quiz / Knowledge Check

### Load Balancing Questions

1. **What type of load balancer would you use for a global HTTP API with SSL termination?**
   - Answer: Global HTTP(S) Load Balancer

2. **How do you ensure backend VMs only receive traffic through the load balancer?**
   - Answer: Configure firewall rules to only allow traffic from load balancer IP ranges (130.211.0.0/22, 35.191.0.0/16)

3. **What's the difference between session affinity and stateless architecture?**
   - Answer: Session affinity routes requests from same client to same backend; stateless architecture allows any backend to handle any request

### Compute Engine Questions

1. **When would you choose preemptible VMs over regular VMs?**
   - Answer: For fault-tolerant, batch processing workloads where cost savings (up to 80%) outweigh the risk of interruption

2. **What's the benefit of using instance templates?**
   - Answer: Consistent VM configuration, easier scaling, simplified management, and integration with MIGs

3. **How do sustained use discounts differ from committed use discounts?**
   - Answer: Sustained use discounts are automatic (no commitment) based on usage; committed use discounts require 1-3 year commitment for higher savings

### Architecture Questions

1. **Design a highly available web application architecture in GCP.**
   - Answer: Multi-region MIGs behind global load balancer, with Cloud SQL in HA configuration, Cloud Storage for static assets, and Cloud CDN

2. **How would you implement a blue-green deployment using GCP services?**
   - Answer: Use MIGs with different instance templates, update URL map to switch traffic between backend services

3. **What security measures should be implemented for production workloads?**
   - Answer: VPC with custom subnets, restrictive firewall rules, service accounts with minimal permissions, OS Login, regular patching, and monitoring

---

## Additional Resources

### GCP Documentation Links
- [Compute Engine Documentation](https://cloud.google.com/compute/docs)
- [Load Balancing Documentation](https://cloud.google.com/load-balancing/docs)
- [VPC Documentation](https://cloud.google.com/vpc/docs)
- [IAM Documentation](https://cloud.google.com/iam/docs)

### Best Practice Guides
- [Compute Engine Best Practices](https://cloud.google.com/compute/docs/best-practices)
- [Load Balancing Best Practices](https://cloud.google.com/load-balancing/docs/best-practices)
- [Security Best Practices](https://cloud.google.com/security/best-practices)

### Cost Management Tools
- [GCP Pricing Calculator](https://cloud.google.com/products/calculator)
- [Cost Management Documentation](https://cloud.google.com/cost-management)
- [Recommender Documentation](https://cloud.google.com/recommender/docs)

---

*This study guide covers the essential concepts for GCP Compute Engine and Load Balancing. Practice with hands-on labs and real-world scenarios to reinforce your understanding.*
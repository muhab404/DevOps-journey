# GCP VPC and Kubernetes (GKE) Integration Guide

## Table of Contents
1. [VPC Fundamentals](#vpc-fundamentals)
2. [VPC Integration with Kubernetes (GKE)](#vpc-integration-with-kubernetes-gke)
3. [VPC Types and Configurations](#vpc-types-and-configurations)
4. [Private Google Access](#private-google-access)
5. [VPC Flow Logs](#vpc-flow-logs)
6. [VPC Routing](#vpc-routing)
7. [Internet Gateway and Cloud NAT](#internet-gateway-and-cloud-nat)
8. [Firewall Rules](#firewall-rules)
9. [Shared VPC](#shared-vpc)
10. [VPC Peering](#vpc-peering)
11. [Cloud VPN](#cloud-vpn)
12. [Cloud Interconnect](#cloud-interconnect)
13. [Best Practices](#best-practices)

## VPC Fundamentals

### What is VPC?
Virtual Private Cloud (VPC) is a virtual network that provides connectivity for your compute resources including GKE clusters, Compute Engine instances, and other Google Cloud services.

### Key VPC Components
- **Subnets**: Regional IP address ranges
- **Routes**: Define paths for network traffic
- **Firewall Rules**: Control traffic flow
- **Gateways**: Provide internet and VPN connectivity

## VPC Types and Configurations

### 1. Auto Mode VPC
Automatically creates subnets in each region with predefined IP ranges.

```bash
# Create auto mode VPC
gcloud compute networks create auto-vpc \
    --subnet-mode=auto \
    --bgp-routing-mode=regional
```

### 2. Custom Mode VPC
Allows manual subnet creation with custom IP ranges.

```bash
# Create custom mode VPC
gcloud compute networks create custom-vpc \
    --subnet-mode=custom \
    --bgp-routing-mode=global

# Create custom subnet
gcloud compute networks subnets create custom-subnet \
    --network=custom-vpc \
    --range=10.1.0.0/16 \
    --region=us-central1
```

### Public vs Private VPC

#### Public VPC Configuration
```bash
# Public subnet with external IP access
gcloud compute networks subnets create public-subnet \
    --network=custom-vpc \
    --range=10.2.0.0/24 \
    --region=us-central1 \
    --enable-ip-alias
```

#### Private VPC Configuration
```bash
# Private subnet without external IP
gcloud compute networks subnets create private-subnet \
    --network=custom-vpc \
    --range=10.3.0.0/24 \
    --region=us-central1 \
    --enable-private-ip-google-access \
    --enable-ip-alias
```

## VPC Integration with Kubernetes (GKE)

### Key Concepts

#### 1. VPC-Native Clusters
GKE clusters that use alias IP ranges for pods and services, providing better integration with VPC networking.

#### 2. IP Address Management
- **Node IP**: Primary IP range for cluster nodes
- **Pod IP**: Secondary IP range for pods
- **Service IP**: Secondary IP range for services

#### 3. Network Architecture

```
VPC Network
├── Node Subnet (Primary Range)
├── Pod IP Range (Secondary Range)
└── Service IP Range (Secondary Range)
```

### Use Cases

1. **Multi-tier Applications**: Separate frontend, backend, and database tiers
2. **Microservices**: Service-to-service communication within VPC
3. **Hybrid Connectivity**: Connect GKE to on-premises networks
4. **Security Isolation**: Network-level security controls
5. **Multi-region Deployments**: Global load balancing across regions

### Required IAM Roles

```bash
# GKE Service Account permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:service-PROJECT_NUMBER@container-engine-robot.iam.gserviceaccount.com" \
    --role="roles/compute.networkUser"

# User permissions for GKE management
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="user:admin@example.com" \
    --role="roles/container.clusterAdmin"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="user:admin@example.com" \
    --role="roles/compute.networkAdmin"
```

### Step-by-Step GKE VPC Setup

#### Step 1: Create VPC and Subnets

```bash
# Set variables
export PROJECT_ID="your-project-id"
export VPC_NAME="gke-vpc"
export SUBNET_NAME="gke-subnet"
export REGION="us-central1"
export ZONE="us-central1-a"

# Create custom VPC
gcloud compute networks create $VPC_NAME \
    --subnet-mode=custom \
    --bgp-routing-mode=regional

# Create subnet with secondary ranges for GKE
gcloud compute networks subnets create $SUBNET_NAME \
    --network=$VPC_NAME \
    --range=10.0.0.0/24 \
    --region=$REGION \
    --secondary-range=pods=10.1.0.0/16,services=10.2.0.0/20 \
    --enable-private-ip-google-access
```

#### Step 2: Create Firewall Rules

```bash
# Allow internal communication
gcloud compute firewall-rules create $VPC_NAME-allow-internal \
    --network=$VPC_NAME \
    --allow=tcp,udp,icmp \
    --source-ranges=10.0.0.0/8

# Allow SSH access
gcloud compute firewall-rules create $VPC_NAME-allow-ssh \
    --network=$VPC_NAME \
    --allow=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=ssh-allowed

# Allow HTTP/HTTPS
gcloud compute firewall-rules create $VPC_NAME-allow-http \
    --network=$VPC_NAME \
    --allow=tcp:80,tcp:443 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=http-server
```

#### Step 3: Create GKE Cluster

```bash
# Create VPC-native GKE cluster
gcloud container clusters create gke-cluster \
    --network=$VPC_NAME \
    --subnetwork=$SUBNET_NAME \
    --cluster-secondary-range-name=pods \
    --services-secondary-range-name=services \
    --enable-ip-alias \
    --zone=$ZONE \
    --enable-private-nodes \
    --master-ipv4-cidr-block=172.16.0.0/28 \
    --enable-network-policy
```

#### Step 4: Configure Cloud NAT (for private nodes)

```bash
# Create Cloud Router
gcloud compute routers create gke-router \
    --network=$VPC_NAME \
    --region=$REGION

# Create Cloud NAT
gcloud compute routers nats create gke-nat \
    --router=gke-router \
    --region=$REGION \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```

### Kubernetes Manifests for VPC Integration

#### Network Policy Example

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
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
```

#### Multi-tier Application Example

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
      - name: backend
        image: node:16-alpine
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          value: "database-service"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 3000
    targetPort: 3000
```

### Helm Chart for VPC-aware Application

```yaml
# values.yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80

networkPolicy:
  enabled: true
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: frontend
      ports:
      - protocol: TCP
        port: 80

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "app.fullname" . }}
spec:
  podSelector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
  policyTypes:
  - Ingress
  ingress:
  {{- with .Values.networkPolicy.ingress }}
    {{- toYaml . | nindent 2 }}
  {{- end }}
{{- end }}
```

## Private Google Access

Allows instances without external IP addresses to reach Google APIs and services.

### Enable Private Google Access

```bash
# Enable on existing subnet
gcloud compute networks subnets update $SUBNET_NAME \
    --region=$REGION \
    --enable-private-ip-google-access

# Verify configuration
gcloud compute networks subnets describe $SUBNET_NAME \
    --region=$REGION \
    --format="get(privateIpGoogleAccess)"
```

### Private Google Access Routes

```bash
# Create route to Google APIs (automatically created when enabled)
gcloud compute routes list --filter="network:$VPC_NAME"
```

### Testing Private Google Access

```yaml
# test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-private-access
spec:
  containers:
  - name: test
    image: google/cloud-sdk:slim
    command: ["sleep", "3600"]
```

```bash
# Test access to Google APIs
kubectl exec test-private-access -- curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
```

## VPC Flow Logs

Monitor network traffic for security analysis and network optimization.

### Enable Flow Logs

```bash
# Enable flow logs on subnet
gcloud compute networks subnets update $SUBNET_NAME \
    --region=$REGION \
    --enable-flow-logs \
    --logging-aggregation-interval=INTERVAL_5_SEC \
    --logging-flow-sampling=0.5 \
    --logging-metadata=INCLUDE_ALL_METADATA
```

### Flow Logs Configuration

```bash
# Custom flow logs configuration
gcloud compute networks subnets update $SUBNET_NAME \
    --region=$REGION \
    --enable-flow-logs \
    --logging-aggregation-interval=INTERVAL_1_MIN \
    --logging-flow-sampling=1.0 \
    --logging-metadata=EXCLUDE_ALL_METADATA \
    --logging-filter-expr='jsonPayload.src_ip="10.1.0.0/16"'
```

### Query Flow Logs

```sql
-- BigQuery query for flow logs analysis
SELECT
  jsonPayload.connection.src_ip,
  jsonPayload.connection.dest_ip,
  jsonPayload.connection.dest_port,
  jsonPayload.bytes_sent,
  timestamp
FROM `PROJECT_ID.vpc_flows.compute_googleapis_com_vpc_flows_*`
WHERE DATE(_PARTITIONTIME) = CURRENT_DATE()
  AND jsonPayload.connection.dest_port = 443
ORDER BY timestamp DESC
LIMIT 100
```

## VPC Routing

### Default Routes

```bash
# List all routes
gcloud compute routes list --filter="network:$VPC_NAME"

# Default internet gateway route (0.0.0.0/0)
gcloud compute routes describe default-route-internet-gateway \
    --format="table(name,destRange,nextHopGateway,priority)"
```

### Custom Routes

```bash
# Create custom route
gcloud compute routes create custom-route-to-onprem \
    --network=$VPC_NAME \
    --destination-range=192.168.0.0/16 \
    --next-hop-vpn-tunnel=vpn-tunnel-1 \
    --priority=1000

# Route via instance
gcloud compute routes create route-via-instance \
    --network=$VPC_NAME \
    --destination-range=172.16.0.0/12 \
    --next-hop-instance=router-instance \
    --next-hop-instance-zone=$ZONE \
    --priority=1000
```

### Route Priority and Selection

```bash
# Routes with different priorities
gcloud compute routes create high-priority-route \
    --network=$VPC_NAME \
    --destination-range=10.10.0.0/16 \
    --next-hop-gateway=default-internet-gateway \
    --priority=100

gcloud compute routes create low-priority-route \
    --network=$VPC_NAME \
    --destination-range=10.10.0.0/16 \
    --next-hop-vpn-tunnel=backup-tunnel \
    --priority=1000
```

## Internet Gateway and Cloud NAT

### Internet Gateway
Provides internet access for instances with external IP addresses.

```bash
# Internet gateway is automatically available
# Check default route to internet gateway
gcloud compute routes list \
    --filter="network:$VPC_NAME AND nextHopGateway:default-internet-gateway"
```

### Cloud NAT Configuration

#### SNAT (Source NAT)
```bash
# Create Cloud Router
gcloud compute routers create nat-router \
    --network=$VPC_NAME \
    --region=$REGION \
    --asn=65001

# Create Cloud NAT for outbound traffic
gcloud compute routers nats create nat-gateway \
    --router=nat-router \
    --region=$REGION \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips \
    --min-ports-per-vm=64 \
    --max-ports-per-vm=1024
```

#### Advanced NAT Configuration

```bash
# NAT with specific subnets and IP addresses
gcloud compute addresses create nat-ip-1 nat-ip-2 \
    --region=$REGION

gcloud compute routers nats create selective-nat \
    --router=nat-router \
    --region=$REGION \
    --nat-custom-subnet-ip-ranges=subnet-1:ALL_RANGES \
    --nat-external-ip-pool=nat-ip-1,nat-ip-2 \
    --enable-logging \
    --log-filter=ALL
```

#### DNAT (Destination NAT) - Port Forwarding

```bash
# Create forwarding rule for DNAT
gcloud compute forwarding-rules create dnat-rule \
    --region=$REGION \
    --ip-protocol=TCP \
    --ports=80 \
    --target-instance=web-server \
    --target-instance-zone=$ZONE
```

## Firewall Rules

### Stateful Firewall Rules

#### Ingress Rules

```bash
# Allow HTTP ingress
gcloud compute firewall-rules create allow-http-ingress \
    --network=$VPC_NAME \
    --action=ALLOW \
    --direction=INGRESS \
    --rules=tcp:80,tcp:443 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=web-server \
    --priority=1000

# Allow specific source IPs
gcloud compute firewall-rules create allow-admin-ssh \
    --network=$VPC_NAME \
    --action=ALLOW \
    --direction=INGRESS \
    --rules=tcp:22 \
    --source-ranges=203.0.113.0/24 \
    --target-service-accounts=admin-sa@project.iam.gserviceaccount.com \
    --priority=500
```

#### Egress Rules

```bash
# Deny all egress (override default allow)
gcloud compute firewall-rules create deny-all-egress \
    --network=$VPC_NAME \
    --action=DENY \
    --direction=EGRESS \
    --rules=all \
    --destination-ranges=0.0.0.0/0 \
    --priority=1000

# Allow specific egress
gcloud compute firewall-rules create allow-https-egress \
    --network=$VPC_NAME \
    --action=ALLOW \
    --direction=EGRESS \
    --rules=tcp:443 \
    --destination-ranges=0.0.0.0/0 \
    --target-tags=internet-access \
    --priority=500
```

### Priority and Tags

```bash
# High priority rule (lower number = higher priority)
gcloud compute firewall-rules create emergency-access \
    --network=$VPC_NAME \
    --action=ALLOW \
    --direction=INGRESS \
    --rules=tcp:22 \
    --source-ranges=198.51.100.0/24 \
    --target-tags=emergency-access \
    --priority=1

# Tag-based rules
gcloud compute firewall-rules create database-access \
    --network=$VPC_NAME \
    --action=ALLOW \
    --direction=INGRESS \
    --rules=tcp:3306 \
    --source-tags=app-server \
    --target-tags=database-server \
    --priority=1000
```

### Service Account-based Rules

```bash
# Rules based on service accounts
gcloud compute firewall-rules create sa-based-rule \
    --network=$VPC_NAME \
    --action=ALLOW \
    --direction=INGRESS \
    --rules=tcp:8080 \
    --source-service-accounts=frontend-sa@project.iam.gserviceaccount.com \
    --target-service-accounts=backend-sa@project.iam.gserviceaccount.com
```

## Shared VPC

Allows multiple projects to share a common VPC network.

### Host Project Setup

```bash
# Enable Shared VPC on host project
gcloud compute shared-vpc enable HOST_PROJECT_ID

# Associate service project
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
    --host-project=HOST_PROJECT_ID
```

### Service Project Configuration

```bash
# Grant IAM permissions for service project
gcloud projects add-iam-policy-binding HOST_PROJECT_ID \
    --member="serviceAccount:service-SERVICE_PROJECT_NUMBER@container-engine-robot.iam.gserviceaccount.com" \
    --role="roles/compute.networkUser"

# Grant subnet-level permissions
gcloud compute networks subnets add-iam-policy-binding SUBNET_NAME \
    --region=REGION \
    --member="serviceAccount:service-SERVICE_PROJECT_NUMBER@container-engine-robot.iam.gserviceaccount.com" \
    --role="roles/compute.networkUser"
```

### GKE in Shared VPC

```bash
# Create GKE cluster in service project using host project VPC
gcloud container clusters create shared-vpc-cluster \
    --project=SERVICE_PROJECT_ID \
    --network=projects/HOST_PROJECT_ID/global/networks/shared-vpc \
    --subnetwork=projects/HOST_PROJECT_ID/regions/us-central1/subnetworks/gke-subnet \
    --cluster-secondary-range-name=pods \
    --services-secondary-range-name=services \
    --enable-ip-alias \
    --zone=us-central1-a
```

### Auto Import/Export (Not Transitive)

```bash
# Shared VPC does not support route import/export
# Routes are not automatically shared between projects
# Each project must configure its own routes to shared resources
```

## VPC Peering

Connects two VPC networks, allowing private communication between them.

### Create VPC Peering

```bash
# Create peering from VPC-A to VPC-B
gcloud compute networks peerings create peer-a-to-b \
    --network=vpc-a \
    --peer-project=PROJECT_B \
    --peer-network=vpc-b \
    --auto-create-routes

# Create reverse peering from VPC-B to VPC-A
gcloud compute networks peerings create peer-b-to-a \
    --network=vpc-b \
    --peer-project=PROJECT_A \
    --peer-network=vpc-a \
    --auto-create-routes
```

### Peering with Route Import/Export

```bash
# Enable custom route import/export
gcloud compute networks peerings update peer-a-to-b \
    --network=vpc-a \
    --import-custom-routes \
    --export-custom-routes

gcloud compute networks peerings update peer-b-to-a \
    --network=vpc-b \
    --import-custom-routes \
    --export-custom-routes
```

### Peering Limitations

```bash
# Peering is not transitive
# VPC-A <-> VPC-B <-> VPC-C does not mean VPC-A <-> VPC-C
# Maximum 25 peering connections per VPC
# Subnet ranges cannot overlap
```

## Cloud VPN

### VPN Gateway Types

#### Classic VPN (Single tunnel)

```bash
# Create VPN gateway
gcloud compute vpn-gateways create classic-vpn-gateway \
    --network=$VPC_NAME \
    --region=$REGION

# Create VPN tunnel
gcloud compute vpn-tunnels create tunnel-to-onprem \
    --peer-address=203.0.113.12 \
    --shared-secret=shared-secret-key \
    --target-vpn-gateway=classic-vpn-gateway \
    --region=$REGION \
    --ike-version=2
```

#### HA VPN (High Availability)

```bash
# Create HA VPN gateway
gcloud compute vpn-gateways create ha-vpn-gateway \
    --network=$VPC_NAME \
    --region=$REGION

# Create Cloud Router for BGP
gcloud compute routers create vpn-router \
    --network=$VPC_NAME \
    --region=$REGION \
    --asn=65001

# Create VPN tunnels (2 for HA)
gcloud compute vpn-tunnels create tunnel-1 \
    --peer-gcp-gateway=peer-gateway \
    --region=$REGION \
    --ike-version=2 \
    --shared-secret=tunnel-1-secret \
    --router=vpn-router \
    --vpn-gateway=ha-vpn-gateway \
    --interface=0

gcloud compute vpn-tunnels create tunnel-2 \
    --peer-gcp-gateway=peer-gateway \
    --region=$REGION \
    --ike-version=2 \
    --shared-secret=tunnel-2-secret \
    --router=vpn-router \
    --vpn-gateway=ha-vpn-gateway \
    --interface=1
```

### BGP Configuration

```bash
# Create BGP sessions
gcloud compute routers add-bgp-peer vpn-router \
    --peer-name=bgp-peer-1 \
    --interface=tunnel-1-interface \
    --peer-ip-address=169.254.1.2 \
    --peer-asn=65002 \
    --region=$REGION

gcloud compute routers add-bgp-peer vpn-router \
    --peer-name=bgp-peer-2 \
    --interface=tunnel-2-interface \
    --peer-ip-address=169.254.2.2 \
    --peer-asn=65002 \
    --region=$REGION
```

### Static Routes vs Dynamic Routes (BGP)

#### Static Routes

```bash
# Create static route for VPN
gcloud compute routes create route-to-onprem \
    --network=$VPC_NAME \
    --destination-range=192.168.0.0/16 \
    --next-hop-vpn-tunnel=tunnel-to-onprem \
    --priority=1000
```

#### Dynamic Routes (BGP)

```bash
# BGP automatically exchanges routes
# Configure route advertisement
gcloud compute routers update-bgp-peer vpn-router \
    --peer-name=bgp-peer-1 \
    --region=$REGION \
    --advertised-route-priority=100 \
    --advertisement-mode=CUSTOM \
    --set-advertisement-ranges=10.0.0.0/8:100
```

### VPN with AWS

#### AWS Side Configuration

```bash
# AWS Virtual Private Gateway
aws ec2 create-vpn-gateway --type ipsec.1

# AWS Customer Gateway (GCP side)
aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip GCP_VPN_GATEWAY_IP \
    --bgp-asn 65001

# AWS Site-to-Site VPN Connection
aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id cgw-xxx \
    --vpn-gateway-id vgw-xxx
```

#### GCP Side Configuration for AWS VPN

```bash
# Create external VPN gateway for AWS
gcloud compute external-vpn-gateways create aws-vpn-gateway \
    --interfaces 0=AWS_TUNNEL_1_IP,1=AWS_TUNNEL_2_IP

# Create VPN tunnels to AWS
gcloud compute vpn-tunnels create tunnel-to-aws-1 \
    --peer-external-gateway=aws-vpn-gateway \
    --peer-external-gateway-interface=0 \
    --region=$REGION \
    --ike-version=1 \
    --shared-secret=AWS_TUNNEL_1_PSK \
    --router=vpn-router \
    --vpn-gateway=ha-vpn-gateway \
    --interface=0

gcloud compute vpn-tunnels create tunnel-to-aws-2 \
    --peer-external-gateway=aws-vpn-gateway \
    --peer-external-gateway-interface=1 \
    --region=$REGION \
    --ike-version=1 \
    --shared-secret=AWS_TUNNEL_2_PSK \
    --router=vpn-router \
    --vpn-gateway=ha-vpn-gateway \
    --interface=1
```

## Cloud Interconnect

### Dedicated Interconnect

Direct physical connection between your on-premises network and Google's network.

```bash
# Create interconnect attachment
gcloud compute interconnects attachments dedicated create my-attachment \
    --region=$REGION \
    --router=interconnect-router \
    --interconnect=my-interconnect \
    --vlan=100

# Create Cloud Router for interconnect
gcloud compute routers create interconnect-router \
    --network=$VPC_NAME \
    --region=$REGION \
    --asn=65001
```

### Partner Interconnect

Connection through a supported service provider.

```bash
# Create partner attachment
gcloud compute interconnects attachments partner create partner-attachment \
    --region=$REGION \
    --router=interconnect-router \
    --edge-availability-domain=AVAILABILITY_DOMAIN_1

# Configure BGP session
gcloud compute routers add-bgp-peer interconnect-router \
    --peer-name=partner-bgp-peer \
    --interface=partner-interface \
    --peer-ip-address=169.254.1.2 \
    --peer-asn=65002 \
    --region=$REGION
```

### Cross-Cloud Interconnect

Connect to other cloud providers through Google's network.

```bash
# Create cross-cloud attachment
gcloud compute interconnects attachments partner create cross-cloud-attachment \
    --region=$REGION \
    --router=cross-cloud-router \
    --edge-availability-domain=AVAILABILITY_DOMAIN_1 \
    --type=PARTNER \
    --partner-metadata=partner-name=aws,partner-portal-url=https://aws.amazon.com
```

## Best Practices

### Network Design

#### 1. IP Address Planning

```bash
# Use non-overlapping CIDR blocks
# Primary subnet: 10.0.0.0/24 (nodes)
# Secondary ranges: 
#   - Pods: 10.1.0.0/16
#   - Services: 10.2.0.0/20

# Reserve IP ranges for future expansion
gcloud compute addresses create reserved-range \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --network=$VPC_NAME
```

#### 2. Subnet Design

```bash
# Regional subnets for high availability
gcloud compute networks subnets create multi-zone-subnet \
    --network=$VPC_NAME \
    --range=10.3.0.0/24 \
    --region=$REGION \
    --secondary-range=pods=10.4.0.0/16,services=10.5.0.0/20 \
    --enable-private-ip-google-access \
    --enable-flow-logs
```

### Security Best Practices

#### 1. Network Segmentation

```yaml
# Namespace-based network policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: production
```

#### 2. Firewall Rules Hierarchy

```bash
# Deny all by default (highest priority)
gcloud compute firewall-rules create deny-all-default \
    --network=$VPC_NAME \
    --action=DENY \
    --direction=INGRESS \
    --rules=all \
    --source-ranges=0.0.0.0/0 \
    --priority=65534

# Allow specific services (lower priority numbers)
gcloud compute firewall-rules create allow-gke-nodes \
    --network=$VPC_NAME \
    --action=ALLOW \
    --direction=INGRESS \
    --rules=tcp:10250,tcp:443 \
    --source-ranges=10.0.0.0/8 \
    --target-tags=gke-node \
    --priority=1000
```

#### 3. Private Clusters

```bash
# Create private GKE cluster
gcloud container clusters create private-cluster \
    --network=$VPC_NAME \
    --subnetwork=$SUBNET_NAME \
    --enable-private-nodes \
    --enable-private-endpoint \
    --master-ipv4-cidr-block=172.16.0.0/28 \
    --enable-ip-alias \
    --cluster-secondary-range-name=pods \
    --services-secondary-range-name=services
```

### Performance Optimization

#### 1. Regional Persistent Disks

```yaml
# Regional SSD for high availability
apiVersion: v1
kind: StorageClass
metadata:
  name: regional-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
  zones: us-central1-a,us-central1-b
```

#### 2. Load Balancer Optimization

```yaml
# Internal load balancer
apiVersion: v1
kind: Service
metadata:
  name: internal-service
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

### Cost Considerations

#### 1. Network Egress Costs

```bash
# Use regional resources to minimize egress
# Keep traffic within the same region when possible
# Use Cloud CDN for static content

# Monitor egress with labels
gcloud compute instances create cost-optimized-vm \
    --zone=$ZONE \
    --machine-type=e2-micro \
    --network-interface=subnet=$SUBNET_NAME,no-address \
    --labels=cost-center=development,team=backend
```

#### 2. NAT Gateway Costs

```bash
# Optimize NAT configuration
gcloud compute routers nats update nat-gateway \
    --router=nat-router \
    --region=$REGION \
    --min-ports-per-vm=32 \
    --udp-idle-timeout=30s \
    --tcp-established-idle-timeout=1200s \
    --tcp-transitory-idle-timeout=30s
```

#### 3. Interconnect vs VPN Cost Analysis

```bash
# VPN: $0.05/hour per tunnel + egress charges
# Dedicated Interconnect: $1,700/month per 10 Gbps + port hours
# Partner Interconnect: Varies by partner + port hours

# Calculate breakeven point based on bandwidth requirements
# Use VPN for < 1 Gbps sustained traffic
# Use Interconnect for > 1 Gbps sustained traffic
```

### Monitoring and Troubleshooting

#### 1. VPC Flow Logs Analysis

```sql
-- Top talkers query
SELECT
  jsonPayload.connection.src_ip,
  jsonPayload.connection.dest_ip,
  SUM(CAST(jsonPayload.bytes_sent AS INT64)) as total_bytes
FROM `PROJECT_ID.vpc_flows.compute_googleapis_com_vpc_flows_*`
WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY 1, 2
ORDER BY total_bytes DESC
LIMIT 100
```

#### 2. Network Connectivity Testing

```yaml
# Network debugging pod
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
```

```bash
# Test connectivity
kubectl exec network-debug -- ping 8.8.8.8
kubectl exec network-debug -- nslookup kubernetes.default.svc.cluster.local
kubectl exec network-debug -- curl -I http://backend-service:8080
```

#### 3. Firewall Rules Testing

```bash
# Test firewall rules
gcloud compute firewall-rules list --filter="network:$VPC_NAME"

# Simulate firewall rule evaluation
gcloud compute networks get-effective-firewalls $VPC_NAME \
    --region=$REGION
```

This comprehensive guide covers VPC integration with Kubernetes (GKE) and all the networking concepts you requested. The examples provided are practical and can be adapted to your specific use cases. Remember to always follow security best practices and monitor your network costs regularly.
# GCP Networking Complete Guide

## Table of Contents
1. [VPC Types](#vpc-types)
2. [Public and Private VPC](#public-and-private-vpc)
3. [Private Google Access](#private-google-access)
4. [Flow Logs](#flow-logs)
5. [VPC Routing](#vpc-routing)
6. [Internet Gateway](#internet-gateway)
7. [Cloud NAT](#cloud-nat)
8. [Firewall Rules](#firewall-rules)
9. [Shared VPC](#shared-vpc)
10. [VPC Peering](#vpc-peering)
11. [Cloud VPN](#cloud-vpn)
12. [GCP-AWS VPN Connection](#gcp-aws-vpn-connection)
13. [Cloud Interconnect](#cloud-interconnect)

## VPC Types

### Default VPC
- Automatically created with predefined subnets in each region
- One subnet per region with predetermined IP ranges
- Includes default firewall rules

### Custom VPC
- User-defined with custom subnets and configurations
- Full control over IP ranges and subnet placement
- No default subnets created

**Example:**
```bash
# Create custom VPC
gcloud compute networks create my-custom-vpc --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create my-subnet \
    --network=my-custom-vpc \
    --range=10.0.1.0/24 \
    --region=us-central1
```

## Public and Private VPC

### Public Subnet
- Has route to internet gateway (0.0.0.0/0)
- VMs can have external IP addresses
- Direct internet connectivity

### Private Subnet
- No direct internet access
- Uses NAT for outbound connectivity
- Enhanced security for internal resources

**Example:**
```bash
# Public subnet (default route exists)
gcloud compute instances create public-vm \
    --subnet=public-subnet \
    --address=external-ip

# Private subnet (no external IP)
gcloud compute instances create private-vm \
    --subnet=private-subnet \
    --no-address
```

## Private Google Access

Allows VMs without external IPs to access Google APIs and services through Google's internal network.

**Benefits:**
- Secure access to Google services
- No external IP required
- Reduced attack surface

**Example:**
```bash
# Enable Private Google Access on subnet
gcloud compute networks subnets update my-subnet \
    --enable-private-ip-google-access \
    --region=us-central1
```

## Flow Logs

Network monitoring feature for troubleshooting and security analysis.

**Use Cases:**
- Network troubleshooting
- Security analysis
- Performance monitoring
- Compliance auditing

**Example:**
```bash
# Enable flow logs
gcloud compute networks subnets update my-subnet \
    --enable-flow-logs \
    --logging-flow-sampling=0.1 \
    --logging-aggregation-interval=interval-5-sec \
    --region=us-central1
```

## VPC Routing

### System Routes
- Automatically created by GCP
- Subnet routes (local traffic)
- Default route (0.0.0.0/0 to internet gateway)

### Custom Routes
- User-defined static or dynamic routes
- Override system routes with higher priority
- Support multiple next-hop types

**Example:**
```bash
# Create custom route
gcloud compute routes create my-route \
    --network=my-vpc \
    --destination-range=192.168.1.0/24 \
    --next-hop-instance=my-instance \
    --next-hop-instance-zone=us-central1-a \
    --priority=100
```

## Internet Gateway

In GCP, internet gateway functionality is implicit:
- VMs with external IPs can reach internet through default route
- No separate internet gateway resource to manage
- Automatically handles NAT for external IP addresses

## Cloud NAT

Provides outbound internet access for private instances without external IPs.

### NAT Types
- **SNAT (Source NAT)**: Outbound traffic - translates private IP to public IP
- **DNAT (Destination NAT)**: Inbound traffic - translates public IP to private IP

**Example:**
```bash
# Create Cloud Router
gcloud compute routers create my-router \
    --network=my-vpc \
    --region=us-central1

# Create NAT gateway
gcloud compute routers nats create my-nat \
    --router=my-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips \
    --region=us-central1
```

## Firewall Rules

### Key Characteristics
- **Stateful**: Return traffic automatically allowed
- **Ingress**: Inbound traffic rules
- **Egress**: Outbound traffic rules
- **Priority**: Lower numbers = higher priority (0-65535)
- **Tags**: Apply rules to specific instances

### Rule Components
- Direction (ingress/egress)
- Action (allow/deny)
- Targets (tags, service accounts, IP ranges)
- Sources/Destinations
- Protocols and ports

**Examples:**
```bash
# Ingress rule - Allow SSH
gcloud compute firewall-rules create allow-ssh \
    --network=my-vpc \
    --action=allow \
    --rules=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=ssh-server \
    --priority=1000

# Egress rule - Deny all outbound
gcloud compute firewall-rules create deny-all-egress \
    --network=my-vpc \
    --action=deny \
    --rules=all \
    --direction=egress \
    --priority=1000

# Allow internal communication
gcloud compute firewall-rules create allow-internal \
    --network=my-vpc \
    --action=allow \
    --rules=all \
    --source-ranges=10.0.0.0/8 \
    --priority=500
```

## Shared VPC

Allows multiple projects to share a common VPC network.

### Components
- **Host Project**: Contains the shared VPC network
- **Service Project**: Uses shared VPC resources
- **Auto Import/Export**: Routes are not transitive between service projects

### Benefits
- Centralized network management
- Resource sharing across projects
- Simplified billing and administration

**Example:**
```bash
# Enable shared VPC on host project
gcloud compute shared-vpc enable HOST_PROJECT_ID

# Associate service project
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
    --host-project=HOST_PROJECT_ID

# Grant IAM permissions
gcloud projects add-iam-policy-binding SERVICE_PROJECT_ID \
    --member="serviceAccount:service-account@HOST_PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/compute.networkUser"
```

## VPC Peering

Connects VPC networks across projects or organizations using private IP addresses.

### Characteristics
- **Non-transitive**: If A peers with B, and B peers with C, A cannot reach C
- **Private connectivity**: Uses internal IP addresses
- **Cross-project**: Can peer VPCs in different projects
- **Automatic route exchange**: Routes are automatically exchanged

**Example:**
```bash
# Create peering from VPC A to VPC B
gcloud compute networks peerings create peer-a-to-b \
    --network=vpc-a \
    --peer-project=project-b \
    --peer-network=vpc-b \
    --auto-create-routes

# Create reverse peering from VPC B to VPC A
gcloud compute networks peerings create peer-b-to-a \
    --network=vpc-b \
    --peer-project=project-a \
    --peer-network=vpc-a \
    --auto-create-routes
```

## Cloud VPN

Securely connects on-premises networks to GCP VPC through IPSec tunnels.

### Key Concepts
- **BGP (Border Gateway Protocol)**: Dynamic routing protocol
- **ASN (Autonomous System Number)**: Unique identifier for routing domains
- **IPSec**: Encryption protocol for secure tunnels
- **Static Routes**: Manual route configuration
- **Dynamic Routes**: BGP-based automatic routing
- **Single**: One tunnel (lower availability)
- **HA (High Availability)**: Multiple tunnels for redundancy

### VPN Types
1. **Classic VPN**: Legacy, single tunnel
2. **HA VPN**: Recommended, 99.99% SLA with two tunnels

**Example - Static Routing:**
```bash
# Create VPN gateway
gcloud compute vpn-gateways create my-vpn-gateway \
    --network=my-vpc \
    --region=us-central1

# Create tunnel with static routing
gcloud compute vpn-tunnels create my-tunnel \
    --peer-address=203.0.113.1 \
    --shared-secret=my-secret \
    --target-vpn-gateway=my-vpn-gateway \
    --region=us-central1

# Create route for tunnel
gcloud compute routes create tunnel-route \
    --network=my-vpc \
    --destination-range=192.168.1.0/24 \
    --next-hop-vpn-tunnel=my-tunnel \
    --next-hop-vpn-tunnel-region=us-central1
```

**Example - Dynamic Routing (BGP):**
```bash
# Create Cloud Router
gcloud compute routers create my-router \
    --network=my-vpc \
    --asn=65001 \
    --region=us-central1

# Create HA VPN Gateway
gcloud compute vpn-gateways create ha-vpn-gateway \
    --network=my-vpc \
    --region=us-central1

# Create VPN tunnel with BGP
gcloud compute vpn-tunnels create bgp-tunnel \
    --peer-external-gateway=peer-gateway \
    --shared-secret=my-secret \
    --router=my-router \
    --vpn-gateway=ha-vpn-gateway \
    --region=us-central1
```

## GCP-AWS VPN Connection

Establishes secure connectivity between GCP and AWS using VPN.

### GCP Components
- **VPN Gateway**: GCP's VPN endpoint
- **Cloud Router**: Handles BGP routing
- **Peer Gateway**: Represents AWS endpoint

### AWS Components
- **Virtual Gateway/Transit Gateway**: AWS VPN endpoint
- **Customer Gateway**: Represents GCP endpoint in AWS
- **Site-to-Site VPN**: AWS VPN connection type

### Authentication
- **Pre-shared Key**: Shared secret for tunnel authentication

**Example:**
```bash
# GCP side - Create HA VPN
gcloud compute vpn-gateways create gcp-to-aws-gateway \
    --network=my-vpc \
    --region=us-central1

# Create external peer gateway (AWS)
gcloud compute external-vpn-gateways create aws-peer-gateway \
    --interfaces 0=203.0.113.1,1=203.0.113.2

# Create Cloud Router for BGP
gcloud compute routers create gcp-aws-router \
    --network=my-vpc \
    --asn=65001 \
    --region=us-central1

# Create VPN tunnels to AWS
gcloud compute vpn-tunnels create gcp-aws-tunnel-1 \
    --peer-external-gateway=aws-peer-gateway \
    --peer-external-gateway-interface=0 \
    --shared-secret=pre-shared-key-1 \
    --router=gcp-aws-router \
    --vpn-gateway=gcp-to-aws-gateway \
    --vpn-gateway-interface=0 \
    --region=us-central1

gcloud compute vpn-tunnels create gcp-aws-tunnel-2 \
    --peer-external-gateway=aws-peer-gateway \
    --peer-external-gateway-interface=1 \
    --shared-secret=pre-shared-key-2 \
    --router=gcp-aws-router \
    --vpn-gateway=gcp-to-aws-gateway \
    --vpn-gateway-interface=1 \
    --region=us-central1
```

## Cloud Interconnect

Provides high-bandwidth, low-latency connections between on-premises and GCP.

### Types

#### Dedicated Interconnect
- **Direct physical connection** to Google's network
- **Bandwidth**: 10 Gbps or 100 Gbps
- **Location**: Google's edge points of presence
- **Use case**: High bandwidth requirements, predictable traffic

#### Partner Interconnect
- **Connection through service provider**
- **Bandwidth**: 50 Mbps to 50 Gbps
- **Location**: Service provider facilities
- **Use case**: Lower bandwidth needs, no direct Google presence

#### Cross-Cloud Interconnect
- **Connect to other cloud providers**
- **Supported**: AWS, Azure, Oracle Cloud
- **Benefits**: Multi-cloud connectivity, reduced egress costs

### Components
- **Interconnect**: Physical connection
- **VLAN Attachment**: Layer 2 connection to VPC
- **Cloud Router**: Handles BGP routing
- **BGP Session**: Dynamic routing protocol

**Example:**
```bash
# Create interconnect attachment
gcloud compute interconnects attachments create my-attachment \
    --interconnect=my-interconnect \
    --router=my-router \
    --region=us-central1 \
    --vlan=100

# Create Cloud Router for interconnect
gcloud compute routers create interconnect-router \
    --network=my-vpc \
    --asn=65001 \
    --region=us-central1

# Add BGP peer
gcloud compute routers add-bgp-peer interconnect-router \
    --peer-name=on-prem-peer \
    --peer-asn=65002 \
    --interface=interconnect-interface \
    --peer-ip-address=169.254.1.2 \
    --region=us-central1
```

## Best Practices Summary

### Security
- Use private subnets for internal resources
- Implement least-privilege firewall rules
- Enable Private Google Access for secure API access
- Use service accounts instead of user accounts

### Performance
- Choose appropriate regions for low latency
- Use Cloud Interconnect for high-bandwidth needs
- Implement proper routing for optimal traffic flow
- Monitor with flow logs and Cloud Monitoring

### Cost Optimization
- Use Cloud NAT instead of external IPs where possible
- Implement proper network segmentation
- Choose right interconnect type based on bandwidth needs
- Monitor egress costs and optimize data transfer

### High Availability
- Use HA VPN for critical connections
- Implement redundant network paths
- Use multiple zones for resource distribution
- Plan for disaster recovery scenarios
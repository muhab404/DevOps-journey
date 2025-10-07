# 🐧 Linux Administration 2 - Advanced Enterprise Guide for Senior DevOps Engineers
*Advanced System Administration and Enterprise Infrastructure Management*

## Table of Contents
1. [Advanced Networking & Network Services](#advanced-networking--network-services)
2. [High Availability & Clustering](#high-availability--clustering)
3. [Performance Optimization & Tuning](#performance-optimization--tuning)
4. [Enterprise Storage Solutions](#enterprise-storage-solutions)
5. [Advanced Security & Compliance](#advanced-security--compliance)
6. [Virtualization & Container Orchestration](#virtualization--container-orchestration)
7. [Database Administration & Management](#database-administration--management)
8. [Backup & Disaster Recovery](#backup--disaster-recovery)
9. [Enterprise Monitoring & Observability](#enterprise-monitoring--observability)
10. [Infrastructure Automation & Orchestration](#infrastructure-automation--orchestration)
11. [Cloud Integration & Hybrid Infrastructure](#cloud-integration--hybrid-infrastructure)
12. [Troubleshooting Complex Enterprise Issues](#troubleshooting-complex-enterprise-issues)

---

## 🔹 Advanced Networking & Network Services

### Software-Defined Networking (SDN)

**SDN Architecture Principles:**
Traditional networking couples control and data planes, while SDN separates them for centralized management and programmability.

**Control Plane vs Data Plane:**
- **Control Plane**: Makes routing decisions, maintains topology
- **Data Plane**: Forwards packets based on control plane decisions
- **Management Plane**: Configures and monitors network devices

**OpenFlow Protocol:**
Communication protocol between SDN controller and switches:
- **Flow Tables**: Match-action rules for packet processing
- **Flow Entries**: Specific rules with match fields and actions
- **Controller Communication**: Centralized decision making

**Linux Bridge and Open vSwitch:**
```bash
# Traditional Linux Bridge
brctl addbr br0
brctl addif br0 eth0
brctl addif br0 tap0

# Open vSwitch (OVS)
ovs-vsctl add-br ovs-br0
ovs-vsctl add-port ovs-br0 eth0
ovs-vsctl add-port ovs-br0 tap0

# OVS Flow Rules
ovs-ofctl add-flow ovs-br0 "in_port=1,actions=output:2"
ovs-ofctl add-flow ovs-br0 "dl_type=0x0800,nw_dst=192.168.1.0/24,actions=output:3"
```

### Network Function Virtualization (NFV)

**Virtual Network Functions (VNFs):**
Software implementations of network functions traditionally performed by dedicated hardware:

**Common VNFs:**
- **Virtual Firewalls**: Software-based packet filtering
- **Virtual Load Balancers**: Traffic distribution across servers
- **Virtual Routers**: Software-based routing functions
- **Virtual WAN Optimizers**: Bandwidth optimization

**NFV Infrastructure (NFVI):**
- **Compute Resources**: Virtual machines or containers
- **Storage Resources**: Distributed storage systems
- **Network Resources**: Virtual switches and networks

**Service Function Chaining (SFC):**
Ordered sequence of network functions that packets traverse:
```bash
# Example SFC with iptables and tc
# Traffic -> Firewall -> Load Balancer -> Application

# Firewall rules
iptables -t mangle -A PREROUTING -j MARK --set-mark 1

# Traffic control for load balancing
tc qdisc add dev eth0 root handle 1: htb
tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit
tc filter add dev eth0 protocol ip parent 1:0 prio 1 handle 1 fw flowid 1:1
```

### Advanced Routing Protocols

**Border Gateway Protocol (BGP):**
Path vector protocol for inter-domain routing:

**BGP Concepts:**
- **Autonomous Systems (AS)**: Administrative domains
- **BGP Speakers**: Routers running BGP
- **Path Attributes**: Information about routes
- **Route Selection**: Best path algorithm

**OSPF (Open Shortest Path First):**
Link-state routing protocol for intra-domain routing:

**OSPF Areas:**
- **Backbone Area (Area 0)**: Central area connecting all others
- **Stub Areas**: Areas with limited external connectivity
- **NSSA**: Not-So-Stubby Areas with controlled external routes

**Linux Routing with FRR:**
```bash
# FRRouting configuration
vtysh
configure terminal

# OSPF configuration
router ospf
network 192.168.1.0/24 area 0
network 10.0.0.0/8 area 1

# BGP configuration
router bgp 65001
neighbor 192.168.1.1 remote-as 65002
network 10.0.0.0/8
```

### Network Security Architecture

**Zero Trust Network Model:**
Security model that assumes no implicit trust based on network location:

**Core Principles:**
- **Verify Explicitly**: Authenticate and authorize every transaction
- **Least Privilege Access**: Minimize access rights for users and devices
- **Assume Breach**: Design systems assuming compromise

**Network Segmentation Strategies:**
- **Micro-segmentation**: Fine-grained network isolation
- **VLAN Segmentation**: Layer 2 network isolation
- **Subnet Segmentation**: Layer 3 network isolation
- **Application-level Segmentation**: Service mesh security

**Advanced Firewall Configurations:**
```bash
# nftables advanced rules
nft add table inet enterprise_fw
nft add chain inet enterprise_fw input { type filter hook input priority 0 \; }

# Geo-blocking
nft add set inet enterprise_fw blocked_countries { type ipv4_addr \; flags interval \; }
nft add rule inet enterprise_fw input ip saddr @blocked_countries drop

# Rate limiting
nft add rule inet enterprise_fw input tcp dport 22 limit rate 5/minute accept
nft add rule inet enterprise_fw input tcp dport 22 drop

# Application layer filtering
nft add rule inet enterprise_fw input tcp dport 80 ct state new limit rate 100/second accept
```

---

## 🔹 High Availability & Clustering

### Cluster Architecture Patterns

**Active-Active vs Active-Passive:**

**Active-Active Clustering:**
- All nodes actively serve requests
- Load distributed across nodes
- Higher resource utilization
- Complex data consistency requirements

**Active-Passive Clustering:**
- One node active, others standby
- Automatic failover on failure
- Simpler data consistency
- Lower resource utilization

**Split-Brain Prevention:**
Critical issue where cluster nodes lose communication and both become active:

**Prevention Mechanisms:**
- **Quorum**: Majority voting for cluster decisions
- **STONITH**: Shoot The Other Node In The Head
- **Fencing**: Isolate failed nodes from shared resources
- **Witness Nodes**: Tie-breaker nodes for quorum

### Pacemaker and Corosync

**Corosync Cluster Engine:**
Provides cluster membership and messaging services:

**Key Features:**
- **Membership Management**: Track cluster node status
- **Message Passing**: Reliable communication between nodes
- **Quorum Calculation**: Determine cluster operational status
- **Configuration Synchronization**: Distribute cluster configuration

**Pacemaker Resource Manager:**
High-level cluster resource management:

**Resource Types:**
- **Primitive Resources**: Basic services (IP, filesystem, service)
- **Group Resources**: Collection of resources that move together
- **Clone Resources**: Resources running on multiple nodes
- **Master/Slave Resources**: Resources with primary/secondary roles

**Cluster Configuration:**
```bash
# Basic two-node cluster
pcs cluster auth node1 node2
pcs cluster setup --name mycluster node1 node2
pcs cluster start --all

# Resource configuration
pcs resource create webserver apache configfile=/etc/httpd/conf/httpd.conf
pcs resource create webip IPaddr2 ip=192.168.1.100 cidr_netmask=24
pcs resource group add webgroup webip webserver

# Constraints
pcs constraint location webgroup prefers node1=100
pcs constraint colocation add webserver with webip INFINITY
pcs constraint order webip then webserver
```

### Load Balancing Strategies

**Layer 4 vs Layer 7 Load Balancing:**

**Layer 4 (Transport Layer):**
- Routes based on IP and port
- Protocol agnostic
- Higher performance
- Limited routing decisions

**Layer 7 (Application Layer):**
- Routes based on application data
- Content-aware routing
- SSL termination
- More processing overhead

**HAProxy Advanced Configuration:**
```bash
# /etc/haproxy/haproxy.cfg
global
    daemon
    user haproxy
    group haproxy
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog

frontend web_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem
    redirect scheme https if !{ ssl_fc }
    
    # ACLs for routing
    acl is_api path_beg /api/
    acl is_static path_beg /static/
    
    # Backend selection
    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health
    server web1 192.168.1.10:8080 check
    server web2 192.168.1.11:8080 check
    server web3 192.168.1.12:8080 check

backend api_servers
    balance leastconn
    option httpchk GET /api/health
    server api1 192.168.1.20:8080 check
    server api2 192.168.1.21:8080 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
```

### Database Clustering

**MySQL/MariaDB Galera Cluster:**
Synchronous multi-master replication:

**Galera Features:**
- **Synchronous Replication**: All nodes have identical data
- **Multi-Master**: Write to any node
- **Automatic Node Provisioning**: New nodes sync automatically
- **Conflict Detection**: Handles write conflicts

**PostgreSQL Streaming Replication:**
Asynchronous master-slave replication:

**Replication Types:**
- **Streaming Replication**: Continuous WAL shipping
- **Logical Replication**: Table-level replication
- **Synchronous Replication**: Wait for slave confirmation
- **Hot Standby**: Read-only queries on slaves

**Redis Clustering:**
```bash
# Redis Cluster setup
redis-cli --cluster create \
  192.168.1.10:7000 192.168.1.11:7000 192.168.1.12:7000 \
  192.168.1.10:7001 192.168.1.11:7001 192.168.1.12:7001 \
  --cluster-replicas 1

# Cluster management
redis-cli --cluster check 192.168.1.10:7000
redis-cli --cluster rebalance 192.168.1.10:7000
redis-cli --cluster reshard 192.168.1.10:7000
```

---

## 🔹 Performance Optimization & Tuning

### System Performance Analysis

**Performance Methodology:**
Systematic approach to performance analysis:

**USE Method (Utilization, Saturation, Errors):**
- **Utilization**: Percentage of time resource is busy
- **Saturation**: Degree of queuing or waiting
- **Errors**: Count of error events

**RED Method (Rate, Errors, Duration):**
- **Rate**: Requests per second
- **Errors**: Number of failed requests
- **Duration**: Time to process requests

**Performance Profiling Tools:**
```bash
# CPU profiling
perf record -g ./application
perf report --stdio

# Memory profiling
valgrind --tool=massif ./application
valgrind --tool=memcheck ./application

# I/O profiling
iotop -ao
iostat -x 1

# Network profiling
iftop -i eth0
nethogs eth0
```

### Kernel Performance Tuning

**CPU Scheduling Optimization:**
```bash
# CPU governor settings
echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# CPU affinity for processes
taskset -c 0,1 ./cpu_intensive_app

# IRQ affinity
echo 2 > /proc/irq/24/smp_affinity

# Scheduler tuning
echo 1 > /proc/sys/kernel/sched_autogroup_enabled
echo 10000000 > /proc/sys/kernel/sched_latency_ns
```

**Memory Management Tuning:**
```bash
# Virtual memory settings
echo 10 > /proc/sys/vm/swappiness
echo 1 > /proc/sys/vm/overcommit_memory
echo 80 > /proc/sys/vm/overcommit_ratio

# Huge pages configuration
echo 1024 > /proc/sys/vm/nr_hugepages
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# Memory compaction
echo 1 > /proc/sys/vm/compact_memory
```

**Network Stack Tuning:**
```bash
# TCP buffer sizes
echo 'net.core.rmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 87380 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 134217728' >> /etc/sysctl.conf

# TCP congestion control
echo 'net.ipv4.tcp_congestion_control = bbr' >> /etc/sysctl.conf

# Network device queue tuning
ethtool -G eth0 rx 4096 tx 4096
ethtool -K eth0 gro on gso on tso on
```

### Application Performance Optimization

**JVM Tuning for Java Applications:**
```bash
# Heap sizing
-Xms4g -Xmx8g

# Garbage collection
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m

# JIT compilation
-XX:+TieredCompilation
-XX:TieredStopAtLevel=1

# Monitoring
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
```

**Database Performance Tuning:**

**MySQL/MariaDB Optimization:**
```sql
-- Query cache
SET GLOBAL query_cache_size = 268435456;
SET GLOBAL query_cache_type = ON;

-- Buffer pool
SET GLOBAL innodb_buffer_pool_size = 8589934592;
SET GLOBAL innodb_buffer_pool_instances = 8;

-- I/O optimization
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
SET GLOBAL innodb_flush_method = O_DIRECT;
```

**PostgreSQL Optimization:**
```sql
-- Memory settings
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 256MB

-- Checkpoint settings
checkpoint_completion_target = 0.9
wal_buffers = 16MB

-- Query planner
random_page_cost = 1.1
effective_io_concurrency = 200
```

### Storage Performance Optimization

**File System Tuning:**
```bash
# ext4 optimization
mount -o noatime,nodiratime,data=writeback /dev/sdb1 /data

# XFS optimization
mount -o noatime,nodiratime,logbsize=256k,largeio /dev/sdb1 /data

# I/O scheduler optimization
echo deadline > /sys/block/sdb/queue/scheduler
echo 4096 > /sys/block/sdb/queue/read_ahead_kb
```

**RAID Performance Tuning:**
```bash
# RAID stripe size optimization
mdadm --create /dev/md0 --level=5 --raid-devices=4 --chunk=64 /dev/sd[bcde]1

# RAID write-back cache
echo write-back > /sys/block/md0/md/sync_action

# RAID read-ahead
blockdev --setra 8192 /dev/md0
```

**SSD Optimization:**
```bash
# TRIM support
fstrim -v /
echo 'fstrim -v / >> /var/log/trim.log' | crontab -e

# I/O scheduler for SSD
echo noop > /sys/block/sda/queue/scheduler

# Disable swap on SSD
swapoff -a
```

---

## 🔹 Enterprise Storage Solutions

### Storage Area Networks (SAN)

**SAN Architecture:**
Dedicated high-speed network for storage access:

**SAN Components:**
- **Host Bus Adapters (HBA)**: Connect servers to SAN
- **SAN Switches**: Provide connectivity and routing
- **Storage Arrays**: Centralized storage systems
- **Management Software**: Configuration and monitoring

**Fibre Channel Protocol:**
High-speed network technology for SANs:

**FC Layers:**
- **FC-4**: Protocol mapping (SCSI, IP)
- **FC-3**: Common services
- **FC-2**: Framing and flow control
- **FC-1**: Transmission protocol
- **FC-0**: Physical layer

**iSCSI (Internet Small Computer Systems Interface):**
SCSI over TCP/IP networks:

**iSCSI Components:**
- **Initiator**: Client that initiates SCSI commands
- **Target**: Server that receives SCSI commands
- **Portal**: Network endpoint for iSCSI communication

**Linux iSCSI Configuration:**
```bash
# iSCSI initiator setup
iscsiadm -m discovery -t st -p 192.168.1.100:3260
iscsiadm -m node --targetname iqn.2023-01.com.example:storage --login

# Multipath configuration
multipath -ll
multipathd -k"show paths"

# /etc/multipath.conf
defaults {
    user_friendly_names yes
    path_grouping_policy multibus
    failback immediate
    no_path_retry queue
}
```

### Network Attached Storage (NAS)

**NFS (Network File System):**
Distributed file system protocol:

**NFS Versions:**
- **NFSv3**: Stateless protocol, UDP/TCP support
- **NFSv4**: Stateful protocol, improved security
- **NFSv4.1**: Parallel NFS (pNFS) support
- **NFSv4.2**: Server-side copy, sparse files

**Advanced NFS Configuration:**
```bash
# /etc/exports
/data 192.168.1.0/24(rw,sync,no_root_squash,no_subtree_check)
/home 192.168.1.0/24(rw,sync,root_squash,no_subtree_check)

# NFS performance tuning
echo 'net.core.rmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'sunrpc.tcp_slot_table_entries = 128' >> /etc/sysctl.conf

# Mount options for performance
mount -t nfs -o rsize=32768,wsize=32768,hard,intr,tcp nfs-server:/data /mnt/data
```

**SMB/CIFS Protocol:**
Windows-compatible file sharing:

**Samba Configuration:**
```bash
# /etc/samba/smb.conf
[global]
workgroup = WORKGROUP
server string = Linux File Server
security = user
map to guest = bad user
dns proxy = no

[shared]
path = /srv/samba/shared
browsable = yes
writable = yes
guest ok = yes
read only = no
```

### Distributed Storage Systems

**GlusterFS:**
Scale-out network-attached storage:

**GlusterFS Volume Types:**
- **Distributed**: Files spread across bricks
- **Replicated**: Files mirrored across bricks
- **Striped**: Files striped across bricks
- **Distributed-Replicated**: Combination of distribution and replication

**GlusterFS Setup:**
```bash
# Peer probe
gluster peer probe node2
gluster peer probe node3

# Volume creation
gluster volume create gv0 replica 3 node1:/data/brick1 node2:/data/brick1 node3:/data/brick1
gluster volume start gv0

# Client mount
mount -t glusterfs node1:/gv0 /mnt/gluster
```

**Ceph Storage Cluster:**
Unified storage system providing object, block, and file storage:

**Ceph Components:**
- **MON (Monitor)**: Cluster state and consensus
- **OSD (Object Storage Daemon)**: Data storage
- **MDS (Metadata Server)**: File system metadata
- **RGW (RADOS Gateway)**: Object storage interface

**Ceph Architecture Concepts:**
- **CRUSH Algorithm**: Data placement algorithm
- **Placement Groups**: Logical grouping of objects
- **Pools**: Logical partitions for data
- **RADOS**: Reliable Autonomic Distributed Object Store

### Storage Virtualization

**Logical Volume Management (LVM):**
Advanced LVM features for enterprise storage:

**LVM Snapshots:**
```bash
# Create snapshot
lvcreate -L 10G -s -n backup-snap /dev/vg0/data

# Mount snapshot
mount /dev/vg0/backup-snap /mnt/backup

# Merge snapshot
lvconvert --merge /dev/vg0/backup-snap
```

**LVM Thin Provisioning:**
```bash
# Create thin pool
lvcreate -L 100G --thinpool pool0 vg0

# Create thin volumes
lvcreate -V 50G --thin vg0/pool0 -n thin1
lvcreate -V 50G --thin vg0/pool0 -n thin2

# Monitor thin pool
lvs -a vg0
```

**Device Mapper Multipath:**
Provides redundancy and load balancing for storage paths:

**Multipath Configuration:**
```bash
# /etc/multipath.conf
defaults {
    polling_interval 10
    path_selector "round-robin 0"
    path_grouping_policy multibus
    uid_attribute ID_SERIAL
    rr_min_io 100
    failback immediate
    no_path_retry queue
}

blacklist {
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^hd[a-z]"
}
```

---

## 🔹 Advanced Security & Compliance

### Security Frameworks and Standards

**NIST Cybersecurity Framework:**
Comprehensive approach to cybersecurity risk management:

**Framework Core Functions:**
- **Identify**: Asset management, risk assessment
- **Protect**: Access control, data security
- **Detect**: Anomaly detection, monitoring
- **Respond**: Incident response, communications
- **Recover**: Recovery planning, improvements

**CIS Controls:**
Prioritized set of cybersecurity best practices:

**Critical Security Controls:**
1. **Inventory of Authorized/Unauthorized Devices**
2. **Inventory of Authorized/Unauthorized Software**
3. **Continuous Vulnerability Management**
4. **Controlled Use of Administrative Privileges**
5. **Secure Configuration for Hardware/Software**

**ISO 27001 Implementation:**
Information security management system standard:

**ISMS Components:**
- **Security Policy**: High-level security direction
- **Risk Assessment**: Identify and evaluate risks
- **Risk Treatment**: Implement security controls
- **Monitoring and Review**: Continuous improvement

### Advanced Authentication and Authorization

**LDAP Integration:**
Centralized directory services:

**OpenLDAP Configuration:**
```bash
# /etc/openldap/slapd.conf
database bdb
suffix "dc=example,dc=com"
rootdn "cn=admin,dc=example,dc=com"
rootpw {SSHA}encrypted_password

directory /var/lib/ldap
index objectClass eq
index cn,sn,uid pres,eq,approx,sub
```

**Kerberos Authentication:**
Network authentication protocol:

**Kerberos Components:**
- **KDC (Key Distribution Center)**: Authentication server
- **TGT (Ticket Granting Ticket)**: Initial authentication token
- **Service Tickets**: Access tokens for specific services
- **Principals**: Unique identities in Kerberos realm

**SAML Integration:**
Security Assertion Markup Language for SSO:

**SAML Components:**
- **Identity Provider (IdP)**: Authenticates users
- **Service Provider (SP)**: Provides services
- **Assertions**: Security statements about users
- **Metadata**: Configuration information

### Encryption and Key Management

**Advanced Encryption Standard (AES):**
Symmetric encryption algorithm:

**AES Implementation:**
```bash
# File encryption with GPG
gpg --cipher-algo AES256 --compress-algo 1 --symmetric file.txt

# Disk encryption with LUKS
cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 512 --hash sha512 /dev/sdb1

# Network encryption with stunnel
stunnel /etc/stunnel/stunnel.conf
```

**Public Key Infrastructure (PKI):**
Framework for managing digital certificates:

**PKI Components:**
- **Certificate Authority (CA)**: Issues certificates
- **Registration Authority (RA)**: Verifies identities
- **Certificate Repository**: Stores certificates
- **Certificate Revocation List (CRL)**: Revoked certificates

**Hardware Security Modules (HSM):**
Dedicated cryptographic devices:

**HSM Benefits:**
- **Tamper Resistance**: Physical security
- **Performance**: Hardware acceleration
- **Key Generation**: True random number generation
- **Compliance**: Regulatory requirements

### Security Monitoring and Incident Response

**Security Information and Event Management (SIEM):**
Centralized security monitoring:

**SIEM Capabilities:**
- **Log Collection**: Aggregate logs from multiple sources
- **Correlation**: Identify patterns and relationships
- **Alerting**: Notify on security events
- **Forensics**: Investigate security incidents

**Intrusion Detection Systems (IDS):**
Monitor network and system activities:

**IDS Types:**
- **Network-based IDS (NIDS)**: Monitor network traffic
- **Host-based IDS (HIDS)**: Monitor system activities
- **Signature-based**: Detect known attack patterns
- **Anomaly-based**: Detect deviations from normal behavior

**Suricata Configuration:**
```yaml
# /etc/suricata/suricata.yaml
vars:
  address-groups:
    HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"
    EXTERNAL_NET: "!$HOME_NET"

af-packet:
  - interface: eth0
    cluster-id: 99
    cluster-type: cluster_flow

outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert
        - http
        - dns
        - tls
```

**Incident Response Process:**
Structured approach to security incidents:

**IR Phases:**
1. **Preparation**: Policies, procedures, tools
2. **Identification**: Detect and analyze incidents
3. **Containment**: Limit incident impact
4. **Eradication**: Remove threat from environment
5. **Recovery**: Restore normal operations
6. **Lessons Learned**: Improve processes

---

## 🔹 Virtualization & Container Orchestration

### Advanced Virtualization Technologies

**KVM (Kernel-based Virtual Machine):**
Linux kernel virtualization infrastructure:

**KVM Architecture:**
- **Kernel Module**: Provides virtualization extensions
- **QEMU**: User-space emulator and virtualizer
- **libvirt**: Virtualization management API
- **virt-manager**: Graphical management tool

**Advanced KVM Configuration:**
```bash
# CPU pinning for performance
virsh vcpupin vm-name 0 2
virsh vcpupin vm-name 1 3

# NUMA topology
virsh numatune vm-name --mode strict --nodeset 0

# SR-IOV configuration
echo 4 > /sys/class/net/eth0/device/sriov_numvfs
virsh attach-device vm-name sriov-device.xml
```

**Xen Hypervisor:**
Type-1 bare-metal hypervisor:

**Xen Architecture:**
- **Xen Hypervisor**: Runs directly on hardware
- **Dom0**: Privileged domain with hardware access
- **DomU**: Unprivileged guest domains
- **Paravirtualization**: Modified guest OS for performance

**VMware vSphere Integration:**
Enterprise virtualization platform:

**vSphere Components:**
- **ESXi**: Bare-metal hypervisor
- **vCenter Server**: Centralized management
- **vMotion**: Live migration of VMs
- **DRS**: Distributed Resource Scheduler

### Container Orchestration Platforms

**Kubernetes Advanced Features:**

**Custom Resource Definitions (CRDs):**
Extend Kubernetes API with custom resources:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
              image:
                type: string
```

**Operators Pattern:**
Application-specific controllers for Kubernetes:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-operator
  template:
    metadata:
      labels:
        app: postgres-operator
    spec:
      containers:
      - name: operator
        image: postgres-operator:latest
        env:
        - name: WATCH_NAMESPACE
          value: ""
```

**Service Mesh Architecture:**
Infrastructure layer for service-to-service communication:

**Istio Service Mesh:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

### Container Security

**Container Image Security:**
Secure container image practices:

**Image Scanning:**
```bash
# Trivy vulnerability scanner
trivy image nginx:latest

# Clair vulnerability scanner
clairctl analyze nginx:latest

# Docker Bench Security
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -v /etc:/etc:ro -v /usr/bin/docker-containerd:/usr/bin/docker-containerd:ro \
  -v /usr/bin/docker-runc:/usr/bin/docker-runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security
```

**Runtime Security:**
Protect containers during execution:

**Falco Runtime Security:**
```yaml
# falco_rules.yaml
- rule: Shell in Container
  desc: Notice shell activity within a container
  condition: >
    spawned_process and container and
    (proc.name = bash or proc.name = sh)
  output: >
    Shell spawned in container (user=%user.name container_id=%container.id
    container_name=%container.name shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: WARNING
```

**Pod Security Standards:**
Kubernetes security policies:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

---

## 🔹 Database Administration & Management

### Database High Availability

**MySQL/MariaDB Replication:**
Master-slave and master-master replication:

**Replication Types:**
- **Asynchronous Replication**: Master doesn't wait for slave
- **Semi-synchronous Replication**: Master waits for one slave
- **Synchronous Replication**: Master waits for all slaves

**MySQL Replication Setup:**
```sql
-- Master configuration
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW

-- Create replication user
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

-- Slave configuration
[mysqld]
server-id = 2
relay-log = relay-bin
read-only = 1

-- Start replication
CHANGE MASTER TO
  MASTER_HOST='master-host',
  MASTER_USER='repl',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;
START SLAVE;
```

**PostgreSQL Streaming Replication:**
```bash
# Primary server configuration
# postgresql.conf
wal_level = replica
max_wal_senders = 3
wal_keep_segments = 64
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/archive/%f'

# pg_hba.conf
host replication replicator 192.168.1.0/24 md5

# Standby server setup
pg_basebackup -h primary-host -D /var/lib/postgresql/data -U replicator -P -W

# recovery.conf
standby_mode = 'on'
primary_conninfo = 'host=primary-host port=5432 user=replicator'
trigger_file = '/tmp/postgresql.trigger'
```

### Database Performance Optimization

**Query Optimization:**
Systematic approach to query performance:

**Execution Plan Analysis:**
```sql
-- MySQL
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'user@example.com';

-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'user@example.com';

-- Index optimization
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
```

**Database Partitioning:**
Divide large tables into smaller, manageable pieces:

**Horizontal Partitioning (Sharding):**
```sql
-- Range partitioning
CREATE TABLE orders_2023 PARTITION OF orders
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

-- Hash partitioning
CREATE TABLE users_0 PARTITION OF users
FOR VALUES WITH (MODULUS 4, REMAINDER 0);
```

**Connection Pooling:**
Manage database connections efficiently:

**PgBouncer Configuration:**
```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

### NoSQL Database Management

**MongoDB Replica Sets:**
High availability for MongoDB:
```javascript
// Initialize replica set
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
});

// Add member
rs.add("mongo4:27017");

// Check status
rs.status();
```

**MongoDB Sharding:**
Horizontal scaling for MongoDB:
```javascript
// Enable sharding
sh.enableSharding("mydb");

// Shard collection
sh.shardCollection("mydb.users", { "user_id": 1 });

// Check sharding status
sh.status();
```

**Elasticsearch Cluster:**
Distributed search and analytics:
```yaml
# elasticsearch.yml
cluster.name: production-cluster
node.name: node-1
node.roles: [ master, data, ingest ]
network.host: 0.0.0.0
discovery.seed_hosts: ["node-1", "node-2", "node-3"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# Index template
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "message": { "type": "text" },
        "level": { "type": "keyword" }
      }
    }
  }
}
```

---

## 🔹 Backup & Disaster Recovery

### Backup Strategies

**Backup Types and Strategies:**

**Full Backup:**
Complete copy of all data:
- **Advantages**: Complete data protection, simple restore
- **Disadvantages**: Time-consuming, storage intensive

**Incremental Backup:**
Only changed data since last backup:
- **Advantages**: Fast backup, minimal storage
- **Disadvantages**: Complex restore process

**Differential Backup:**
Changed data since last full backup:
- **Advantages**: Faster restore than incremental
- **Disadvantages**: Growing backup size

**3-2-1 Backup Rule:**
- **3 copies** of important data
- **2 different** storage media types
- **1 offsite** backup location

### Enterprise Backup Solutions

**Bacula Backup System:**
Network backup solution for enterprise environments:

**Bacula Components:**
- **Director**: Controls backup operations
- **Storage Daemon**: Manages backup devices
- **File Daemon**: Runs on client machines
- **Catalog**: Database of backup information

**Bacula Configuration:**
```bash
# bacula-dir.conf
Director {
  Name = backup-dir
  DIRport = 9101
  QueryFile = "/etc/bacula/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/var/run/bacula"
  Maximum Concurrent Jobs = 20
  Password = "director-password"
  Messages = Daemon
}

Job {
  Name = "BackupClient1"
  Type = Backup
  Level = Incremental
  Client = client1-fd
  FileSet = "Full Set"
  Schedule = "WeeklyCycle"
  Storage = File
  Messages = Standard
  Pool = File
  Priority = 10
  Write Bootstrap = "/var/lib/bacula/%c.bsr"
}
```

**Amanda Backup System:**
Advanced Maryland Automatic Network Disk Archiver:

**Amanda Features:**
- **Automatic Scheduling**: Optimizes backup schedules
- **Compression**: Reduces backup size
- **Encryption**: Protects backup data
- **Tape Management**: Handles tape rotation

**Rsync-based Backups:**
Efficient file synchronization:
```bash
#!/bin/bash
# Incremental backup script
BACKUP_SOURCE="/home /etc /var/log"
BACKUP_DEST="/backup/$(hostname)"
BACKUP_DATE=$(date +%Y-%m-%d)

# Create backup directory
mkdir -p "$BACKUP_DEST/$BACKUP_DATE"

# Perform backup with hard links
rsync -av --link-dest="$BACKUP_DEST/latest" \
  $BACKUP_SOURCE "$BACKUP_DEST/$BACKUP_DATE/"

# Update latest symlink
rm -f "$BACKUP_DEST/latest"
ln -s "$BACKUP_DATE" "$BACKUP_DEST/latest"

# Cleanup old backups (keep 30 days)
find "$BACKUP_DEST" -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;
```

### Disaster Recovery Planning

**Recovery Time Objective (RTO):**
Maximum acceptable downtime:
- **Tier 1**: < 1 hour (critical systems)
- **Tier 2**: < 4 hours (important systems)
- **Tier 3**: < 24 hours (standard systems)

**Recovery Point Objective (RPO):**
Maximum acceptable data loss:
- **Tier 1**: < 15 minutes
- **Tier 2**: < 1 hour
- **Tier 3**: < 24 hours

**Disaster Recovery Site Types:**
- **Hot Site**: Fully operational duplicate facility
- **Warm Site**: Partially equipped facility
- **Cold Site**: Basic facility with no equipment

**Database Point-in-Time Recovery:**
```sql
-- MySQL point-in-time recovery
mysqlbinlog --start-datetime="2023-01-01 10:00:00" \
  --stop-datetime="2023-01-01 11:00:00" \
  mysql-bin.000001 | mysql -u root -p

-- PostgreSQL point-in-time recovery
# recovery.conf
restore_command = 'cp /var/lib/postgresql/archive/%f %p'
recovery_target_time = '2023-01-01 10:30:00'
```

### Cloud Backup Integration

**AWS Backup Services:**
```bash
# AWS CLI backup commands
aws s3 sync /data/ s3://backup-bucket/data/ --delete

# Glacier archival
aws s3 cp /backup/archive.tar.gz s3://glacier-bucket/ --storage-class GLACIER

# EBS snapshots
aws ec2 create-snapshot --volume-id vol-12345678 --description "Daily backup"
```

**Azure Backup:**
```bash
# Azure CLI backup
az backup vault create --resource-group myRG --name myVault --location eastus

# VM backup
az backup protection enable-for-vm --resource-group myRG --vault-name myVault --vm myVM --policy-name DefaultPolicy
```

**Google Cloud Backup:**
```bash
# Cloud Storage backup
gsutil -m rsync -r -d /data/ gs://backup-bucket/data/

# Persistent disk snapshots
gcloud compute disks snapshot disk-1 --snapshot-names=snapshot-1 --zone=us-central1-a
```

---

## 🔹 Enterprise Monitoring & Observability

### Comprehensive Monitoring Stack

**The Three Pillars of Observability:**

**Metrics:**
Numerical measurements over time:
- **Counter**: Monotonically increasing values
- **Gauge**: Values that can go up or down
- **Histogram**: Distribution of values
- **Summary**: Quantiles and totals

**Logs:**
Discrete events with timestamps:
- **Structured Logs**: JSON, key-value pairs
- **Unstructured Logs**: Free-form text
- **Application Logs**: Business logic events
- **System Logs**: Infrastructure events

**Traces:**
Request flow through distributed systems:
- **Spans**: Individual operations
- **Trace Context**: Correlation across services
- **Sampling**: Reduce trace volume
- **Baggage**: Cross-cutting concerns

### Prometheus and Grafana

**Advanced Prometheus Configuration:**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    region: 'us-east-1'

rule_files:
  - "alerts/*.yml"
  - "recording_rules/*.yml"

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://example.com
        - https://api.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

**PromQL Advanced Queries:**
```promql
# Rate of HTTP requests
rate(http_requests_total[5m])

# 95th percentile response time
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# Prediction
predict_linear(node_filesystem_free_bytes[1h], 4*3600) < 0
```

**Grafana Dashboard as Code:**
```json
{
  "dashboard": {
    "title": "Application Performance",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{status}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec",
            "min": 0
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

### ELK Stack (Elasticsearch, Logstash, Kibana)

**Logstash Pipeline Configuration:**
```ruby
# logstash.conf
input {
  beats {
    port => 5044
  }
  
  syslog {
    port => 514
  }
}

filter {
  if [fields][log_type] == "nginx" {
    grok {
      match => { "message" => "%{NGINXACCESS}" }
    }
    
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
    
    mutate {
      convert => { "response" => "integer" }
      convert => { "bytes" => "integer" }
    }
  }
  
  if [fields][log_type] == "application" {
    json {
      source => "message"
    }
    
    if [level] == "ERROR" {
      mutate {
        add_tag => [ "error" ]
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  if "error" in [tags] {
    email {
      to => "alerts@example.com"
      subject => "Application Error Detected"
      body => "Error: %{message}"
    }
  }
}
```

**Elasticsearch Index Lifecycle Management:**
```json
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "10GB",
            "max_age": "7d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "delete": {
        "min_age": "90d"
      }
    }
  }
}
```

### Distributed Tracing

**Jaeger Tracing System:**
```yaml
# jaeger-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:latest
        ports:
        - containerPort: 16686
        - containerPort: 14268
        env:
        - name: COLLECTOR_ZIPKIN_HTTP_PORT
          value: "9411"
```

**OpenTelemetry Integration:**
```python
# Python application instrumentation
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent",
    agent_port=6831,
)

span_processor = BatchSpanProcessor(jaeger_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Create spans
with tracer.start_as_current_span("database_query") as span:
    span.set_attribute("db.statement", "SELECT * FROM users")
    span.set_attribute("db.type", "postgresql")
    # Database operation here
```

---

## 🔹 Infrastructure Automation & Orchestration

### Advanced Configuration Management

**Ansible Advanced Patterns:**

**Dynamic Inventory:**
```python
#!/usr/bin/env python3
# dynamic_inventory.py
import json
import boto3

def get_inventory():
    ec2 = boto3.client('ec2')
    response = ec2.describe_instances()
    
    inventory = {
        '_meta': {'hostvars': {}},
        'webservers': {'hosts': []},
        'databases': {'hosts': []}
    }
    
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            if instance['State']['Name'] == 'running':
                hostname = instance['PublicDnsName']
                
                # Group by tags
                for tag in instance.get('Tags', []):
                    if tag['Key'] == 'Role':
                        if tag['Value'] == 'web':
                            inventory['webservers']['hosts'].append(hostname)
                        elif tag['Value'] == 'db':
                            inventory['databases']['hosts'].append(hostname)
                
                # Host variables
                inventory['_meta']['hostvars'][hostname] = {
                    'ansible_host': instance['PublicIpAddress'],
                    'instance_id': instance['InstanceId'],
                    'instance_type': instance['InstanceType']
                }
    
    return inventory

if __name__ == '__main__':
    print(json.dumps(get_inventory(), indent=2))
```

**Ansible Vault for Secrets:**
```bash
# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Use in playbook
ansible-playbook --ask-vault-pass playbook.yml
```

**Terraform Advanced Patterns:**

**Module Structure:**
```hcl
# modules/vpc/main.tf
variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

**Remote State Management:**
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "infrastructure/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Data source for remote state
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "terraform-state-bucket"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### CI/CD Pipeline Automation

**GitLab CI/CD Advanced Pipeline:**
```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - deploy-staging
  - deploy-production

variables:
  DOCKER_REGISTRY: registry.gitlab.com
  DOCKER_IMAGE: $DOCKER_REGISTRY/$CI_PROJECT_PATH
  KUBECONFIG: /tmp/kubeconfig

.docker_template: &docker_template
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  <<: *docker_template
  stage: build
  script:
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA
  only:
    - main
    - develop

unit_tests:
  stage: test
  image: node:16
  script:
    - npm install
    - npm run test:unit
  coverage: '/Coverage: \d+\.\d+%/'
  artifacts:
    reports:
      junit: test-results.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

security_scan:
  stage: security
  image: owasp/zap2docker-stable
  script:
    - zap-baseline.py -t $APPLICATION_URL -J zap-report.json
  artifacts:
    reports:
      sast: zap-report.json
  allow_failure: true

deploy_staging:
  stage: deploy-staging
  image: bitnami/kubectl
  script:
    - echo $KUBE_CONFIG | base64 -d > $KUBECONFIG
    - kubectl set image deployment/app app=$DOCKER_IMAGE:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/app -n staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy_production:
  stage: deploy-production
  image: bitnami/kubectl
  script:
    - echo $KUBE_CONFIG | base64 -d > $KUBECONFIG
    - kubectl set image deployment/app app=$DOCKER_IMAGE:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/app -n production
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
```

**Jenkins Pipeline as Code:**
```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${JOB_NAME}"
        KUBECONFIG = credentials('kubeconfig')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                script {
                    def image = docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm install'
                        sh 'npm run test:unit'
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'test-results.xml'
                        }
                    }
                }
                
                stage('Security Scan') {
                    steps {
                        sh 'docker run --rm -v $(pwd):/app -w /app securecodewarrior/docker-security-scan'
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh """
                    kubectl set image deployment/app app=${DOCKER_IMAGE}:${BUILD_NUMBER} -n staging
                    kubectl rollout status deployment/app -n staging
                """
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh """
                    kubectl set image deployment/app app=${DOCKER_IMAGE}:${BUILD_NUMBER} -n production
                    kubectl rollout status deployment/app -n production
                """
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        failure {
            emailext (
                subject: "Build Failed: ${JOB_NAME} - ${BUILD_NUMBER}",
                body: "Build failed. Check console output at ${BUILD_URL}",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
```

### Infrastructure as Code Best Practices

**Immutable Infrastructure:**
Treat infrastructure as disposable and replaceable:

**Benefits:**
- **Consistency**: Identical environments
- **Reliability**: Reduced configuration drift
- **Scalability**: Easy to replicate
- **Security**: Fresh deployments reduce attack surface

**GitOps Workflow:**
Git as single source of truth for infrastructure:

**ArgoCD Configuration:**
```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/k8s-manifests
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

This comprehensive Linux Administration 2 guide provides advanced enterprise-level concepts and practices essential for senior DevOps engineers managing complex infrastructure environments. The focus is on deep understanding of enterprise patterns, scalability considerations, and integration with modern DevOps toolchains.
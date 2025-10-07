# 🏗️ Kubernetes Overview & Architecture - DevOps Interview Guide

## Table of Contents
1. [Kubernetes Overview](#kubernetes-overview)
2. [Core Architecture](#core-architecture)
3. [Control Plane Components](#control-plane-components)
4. [Node Components](#node-components)
5. [Networking Architecture](#networking-architecture)
6. [Storage Architecture](#storage-architecture)
7. [Security Architecture](#security-architecture)
8. [Cluster Setup Examples](#cluster-setup-examples)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)

---

## 🔹 Kubernetes Overview

### What is Kubernetes?

**Kubernetes (K8s)** is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.

### Key Problems Kubernetes Solves

```yaml
# Container Management Challenges:
Manual Operations:
  - Manual container deployment and scaling
  - Complex service discovery and load balancing
  - No automated failure recovery
  - Difficult configuration management

Resource Utilization:
  - Poor resource allocation
  - No automatic scaling based on demand
  - Inefficient cluster utilization
  - Manual capacity planning

High Availability:
  - Single points of failure
  - No automated failover
  - Complex disaster recovery
  - Manual health monitoring
```

### Kubernetes Benefits

```yaml
# Kubernetes Solutions:
Automation:
  - Automated deployment and scaling
  - Self-healing capabilities
  - Rolling updates and rollbacks
  - Automated resource management

Scalability:
  - Horizontal and vertical scaling
  - Auto-scaling based on metrics
  - Efficient resource utilization
  - Multi-cloud portability

Reliability:
  - High availability by design
  - Fault tolerance and recovery
  - Health monitoring and alerting
  - Disaster recovery capabilities
```

### Core Concepts

```yaml
# Kubernetes Abstractions:
Cluster: Set of nodes running containerized applications
Node: Physical or virtual machine in the cluster
Pod: Smallest deployable unit (one or more containers)
Service: Stable network endpoint for pods
Deployment: Declarative updates for pods and ReplicaSets
Namespace: Virtual cluster for resource isolation
```

---

## 🔹 Core Architecture

### High-Level Architecture

```yaml
# Kubernetes Cluster Architecture:
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                   │
├─────────────────────────────────────────────────────────┤
│  Control Plane (Master Nodes)                          │
│  ┌─────────────┬─────────────┬─────────────┬─────────┐  │
│  │ API Server  │ Controller  │ Scheduler   │  etcd   │  │
│  │             │ Manager     │             │         │  │
│  └─────────────┴─────────────┴─────────────┴─────────┘  │
├─────────────────────────────────────────────────────────┤
│  Worker Nodes                                           │
│  ┌─────────────────────────────────────────────────────┐│
│  │ Node 1: kubelet + kube-proxy + Container Runtime   ││
│  │ ┌─────────┬─────────┬─────────┬─────────┬─────────┐││
│  │ │  Pod 1  │  Pod 2  │  Pod 3  │  Pod 4  │  Pod 5  │││
│  │ └─────────┴─────────┴─────────┴─────────┴─────────┘││
│  └─────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────┐│
│  │ Node 2: kubelet + kube-proxy + Container Runtime   ││
│  │ ┌─────────┬─────────┬─────────┬─────────┬─────────┐││
│  │ │  Pod 6  │  Pod 7  │  Pod 8  │  Pod 9  │ Pod 10  │││
│  │ └─────────┴─────────┴─────────┴─────────┴─────────┘││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### Master-Worker Architecture

```yaml
# Control Plane (Master):
Responsibilities:
  - Cluster management and coordination
  - API endpoint for all operations
  - Scheduling decisions
  - Storing cluster state
  - Policy enforcement

# Worker Nodes:
Responsibilities:
  - Running application workloads
  - Container runtime management
  - Network proxy and load balancing
  - Reporting node and pod status
```

---

## 🔹 Control Plane Components

### API Server (kube-apiserver)

```yaml
# API Server Functions:
Primary Interface:
  - REST API for all cluster operations
  - Authentication and authorization
  - Admission control
  - API versioning and validation

Communication Hub:
  - All components communicate through API server
  - Only component that talks to etcd
  - Horizontal scaling support
  - Rate limiting and throttling

# API Server Configuration Example:
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - name: kube-apiserver
    image: k8s.gcr.io/kube-apiserver:v1.28.0
    command:
    - kube-apiserver
    - --advertise-address=192.168.1.100
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --secure-port=6443
    - --service-cluster-ip-range=10.96.0.0/12
```

### etcd

```yaml
# etcd Characteristics:
Distributed Key-Value Store:
  - Stores all cluster state and configuration
  - Consistent and highly available
  - RAFT consensus algorithm
  - Strong consistency guarantees

Data Stored:
  - Cluster configuration
  - Resource definitions (pods, services, etc.)
  - Secrets and ConfigMaps
  - Network policies and RBAC rules

# etcd Configuration:
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  containers:
  - name: etcd
    image: k8s.gcr.io/etcd:3.5.9-0
    command:
    - etcd
    - --advertise-client-urls=https://192.168.1.100:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://192.168.1.100:2380
    - --initial-cluster=master=https://192.168.1.100:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.1.100:2379
    - --listen-peer-urls=https://192.168.1.100:2380
    - --name=master
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

### Controller Manager (kube-controller-manager)

```yaml
# Controller Manager Functions:
Built-in Controllers:
  - Node Controller: Monitors node health
  - Replication Controller: Maintains desired replica count
  - Endpoints Controller: Manages service endpoints
  - Service Account Controller: Creates default service accounts
  - Namespace Controller: Manages namespace lifecycle

Control Loops:
  - Continuously monitors cluster state
  - Takes corrective actions when needed
  - Implements desired state reconciliation
  - Handles resource lifecycle management

# Controller Manager Configuration:
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - name: kube-controller-manager
    image: k8s.gcr.io/kube-controller-manager:v1.28.0
    command:
    - kube-controller-manager
    - --bind-address=127.0.0.1
    - --cluster-cidr=10.244.0.0/16
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --use-service-account-credentials=true
```

### Scheduler (kube-scheduler)

```yaml
# Scheduler Functions:
Pod Placement:
  - Selects optimal node for each pod
  - Considers resource requirements and constraints
  - Implements scheduling policies and priorities
  - Handles affinity and anti-affinity rules

Scheduling Process:
  1. Filtering: Find feasible nodes
  2. Scoring: Rank feasible nodes
  3. Selection: Choose highest scoring node
  4. Binding: Assign pod to selected node

# Scheduler Configuration:
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: k8s.gcr.io/kube-scheduler:v1.28.0
    command:
    - kube-scheduler
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --config=/etc/kubernetes/scheduler-config.yaml

# Custom Scheduler Configuration:
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
      - name: NodeAffinity
      - name: PodTopologySpread
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated
```

---

## 🔹 Node Components

### kubelet

```yaml
# kubelet Functions:
Pod Management:
  - Manages pod lifecycle on the node
  - Pulls container images
  - Starts and stops containers
  - Reports pod and node status

Node Management:
  - Registers node with API server
  - Monitors node health and resources
  - Implements resource limits and QoS
  - Manages volumes and networking

# kubelet Configuration:
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
serverTLSBootstrap: true
```

### kube-proxy

```yaml
# kube-proxy Functions:
Network Proxy:
  - Implements Kubernetes service abstraction
  - Maintains network rules on nodes
  - Performs load balancing for services
  - Handles service discovery

Proxy Modes:
  - iptables: Uses iptables rules (default)
  - ipvs: Uses IP Virtual Server (better performance)
  - userspace: Legacy mode (deprecated)

# kube-proxy Configuration:
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: 0.0.0.0
clientConnection:
  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
clusterCIDR: 10.244.0.0/16
mode: "iptables"
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  minSyncPeriod: 0s
  syncPeriod: 30s
  strictARP: false
```

### Container Runtime

```yaml
# Container Runtime Interface (CRI):
Supported Runtimes:
  - containerd: Industry standard (CNCF project)
  - CRI-O: Lightweight runtime for Kubernetes
  - Docker Engine: Via dockershim (deprecated in 1.24+)

Runtime Responsibilities:
  - Image management (pull, store, delete)
  - Container lifecycle management
  - Container networking setup
  - Resource isolation and limits

# containerd Configuration:
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"

[grpc]
  address = "/run/containerd/containerd.sock"

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "k8s.gcr.io/pause:3.7"
  
[plugins."io.containerd.grpc.v1.cri".containerd]
  snapshotter = "overlayfs"
  default_runtime_name = "runc"
  
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
  
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

---

## 🔹 Networking Architecture

### Cluster Networking Model

```yaml
# Kubernetes Networking Requirements:
Pod-to-Pod Communication:
  - Every pod gets unique IP address
  - Pods can communicate without NAT
  - Flat network namespace across cluster

Service Networking:
  - Services get stable virtual IP (ClusterIP)
  - Load balancing across pod endpoints
  - Service discovery via DNS

External Access:
  - NodePort: Expose service on node ports
  - LoadBalancer: Cloud provider load balancer
  - Ingress: HTTP/HTTPS routing and SSL termination
```

### CNI (Container Network Interface)

```yaml
# Popular CNI Plugins:
Calico:
  - Layer 3 networking with BGP
  - Network policies support
  - IPAM (IP Address Management)
  - High performance and scalability

Flannel:
  - Simple overlay network
  - VXLAN or host-gw backend
  - Easy setup and configuration
  - Good for basic networking needs

Weave Net:
  - Mesh networking
  - Automatic service discovery
  - Network encryption
  - Multi-cloud support

# Calico Configuration Example:
apiVersion: v1
kind: ConfigMap
metadata:
  name: calico-config
  namespace: kube-system
data:
  calico_backend: "bird"
  cluster_type: "k8s,bgp"
  cni_network_config: |
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "datastore_type": "kubernetes",
          "mtu": 1440,
          "ipam": {
            "type": "calico-ipam"
          },
          "policy": {
            "type": "k8s"
          },
          "kubernetes": {
            "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
          }
        }
      ]
    }
```

### Service Mesh Integration

```yaml
# Istio Service Mesh:
Components:
  - Envoy Proxy: Sidecar proxy for traffic management
  - Pilot: Service discovery and configuration
  - Citadel: Certificate management and security
  - Galley: Configuration validation and distribution

Benefits:
  - Traffic management and load balancing
  - Security policies and mTLS
  - Observability and tracing
  - Canary deployments and A/B testing

# Istio Sidecar Injection:
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
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
      containers:
      - name: web
        image: nginx:1.21
        ports:
        - containerPort: 80
```

---

## 🔹 Storage Architecture

### Storage Types

```yaml
# Kubernetes Storage Options:
Ephemeral Storage:
  - emptyDir: Temporary storage tied to pod lifecycle
  - hostPath: Mount host filesystem (not recommended for production)

Persistent Storage:
  - PersistentVolume (PV): Cluster-level storage resource
  - PersistentVolumeClaim (PVC): Request for storage by pod
  - StorageClass: Dynamic provisioning template

Network Storage:
  - NFS: Network File System
  - Ceph: Distributed storage system
  - Cloud Storage: EBS, GCE PD, Azure Disk
```

### Storage Classes and Dynamic Provisioning

```yaml
# AWS EBS StorageClass:
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
# PVC using StorageClass:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: gp3-encrypted
```

### CSI (Container Storage Interface)

```yaml
# CSI Driver Components:
CSI Controller:
  - Handles volume provisioning and attachment
  - Runs as deployment with multiple replicas
  - Implements CreateVolume, DeleteVolume operations

CSI Node:
  - Runs on every node as DaemonSet
  - Handles volume mounting and unmounting
  - Implements NodeStageVolume, NodePublishVolume

# EBS CSI Driver Example:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ebs-csi-controller
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ebs-csi-controller
  template:
    metadata:
      labels:
        app: ebs-csi-controller
    spec:
      serviceAccountName: ebs-csi-controller-sa
      containers:
      - name: ebs-plugin
        image: k8s.gcr.io/provider-aws/aws-ebs-csi-driver:v1.19.0
        args:
        - --endpoint=$(CSI_ENDPOINT)
        - --logtostderr
        - --v=2
        env:
        - name: CSI_ENDPOINT
          value: unix:///var/lib/csi/sockets/pluginproxy/csi.sock
        - name: AWS_REGION
          value: us-west-2
        volumeMounts:
        - name: socket-dir
          mountPath: /var/lib/csi/sockets/pluginproxy/
      volumes:
      - name: socket-dir
        emptyDir: {}
```

---

## 🔹 Security Architecture

### Authentication and Authorization

```yaml
# Authentication Methods:
X.509 Certificates:
  - Client certificates for users and services
  - Certificate-based mutual TLS
  - Certificate rotation and management

Service Account Tokens:
  - JWT tokens for pod authentication
  - Automatic token mounting
  - Token rotation and expiration

External Identity Providers:
  - OIDC (OpenID Connect)
  - LDAP integration
  - Cloud provider IAM

# RBAC Configuration:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-pods
subjects:
- kind: User
  name: jane@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Network Security

```yaml
# Network Policies:
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
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Pod Security

```yaml
# Pod Security Standards:
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Secure Pod Configuration:
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: secure-namespace
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
```

---

## 🔹 Cluster Setup Examples

### kubeadm Cluster Setup

```bash
# Master Node Initialization:
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --apiserver-advertise-address=192.168.1.100 \
  --control-plane-endpoint=k8s-master.local:6443

# Configure kubectl:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI (Flannel):
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Join Worker Nodes:
sudo kubeadm join k8s-master.local:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### High Availability Setup

```yaml
# HA Control Plane with kubeadm:
# Load Balancer Configuration (HAProxy):
global
    log stdout local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option log-health-checks
    option forwardfor
    option http-server-close
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend k8s-api
    bind *:6443
    mode tcp
    default_backend k8s-masters

backend k8s-masters
    mode tcp
    balance roundrobin
    server master1 192.168.1.101:6443 check
    server master2 192.168.1.102:6443 check
    server master3 192.168.1.103:6443 check

# First Master Node:
sudo kubeadm init \
  --control-plane-endpoint="k8s-lb.local:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Additional Master Nodes:
sudo kubeadm join k8s-lb.local:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>
```

### Managed Kubernetes Examples

```yaml
# EKS Cluster (Terraform):
resource "aws_eks_cluster" "main" {
  name     = "production-cluster"
  role_arn = aws_iam_role.cluster.arn
  version  = "1.28"

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = ["0.0.0.0/0"]
  }

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  enabled_cluster_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
  ]
}

# GKE Cluster (Terraform):
resource "google_container_cluster" "primary" {
  name     = "production-cluster"
  location = "us-central1"

  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  ip_allocation_policy {
    cluster_secondary_range_name  = "k8s-pod-range"
    services_secondary_range_name = "k8s-service-range"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  master_auth {
    client_certificate_config {
      issue_client_certificate = false
    }
  }
}
```

---

## 🔹 Best Practices

### Cluster Design

```yaml
# Production Cluster Guidelines:
High Availability:
  - Multiple master nodes (odd number: 3, 5)
  - Separate etcd cluster (external etcd)
  - Load balancer for API server
  - Multi-zone deployment

Security:
  - Enable RBAC and Pod Security Standards
  - Network policies for traffic control
  - Regular security updates and patches
  - Audit logging and monitoring

Resource Management:
  - Resource quotas and limit ranges
  - Node pools for different workload types
  - Cluster autoscaling configuration
  - Monitoring and alerting setup
```

### Operational Excellence

```yaml
# Monitoring and Observability:
Metrics Collection:
  - Prometheus for metrics
  - Grafana for visualization
  - AlertManager for alerting
  - Custom metrics and SLIs

Logging:
  - Centralized logging (ELK, Fluentd)
  - Structured logging format
  - Log retention policies
  - Security and audit logs

Tracing:
  - Distributed tracing (Jaeger, Zipkin)
  - Service mesh integration
  - Performance monitoring
  - Error tracking and debugging
```

### Disaster Recovery

```yaml
# Backup and Recovery:
etcd Backup:
  - Regular automated backups
  - Cross-region backup storage
  - Backup encryption and validation
  - Recovery testing procedures

Application Data:
  - Persistent volume snapshots
  - Database backup strategies
  - Cross-region replication
  - Recovery time objectives (RTO/RPO)

# Backup Script Example:
#!/bin/bash
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Upload to S3
aws s3 cp backup.db s3://k8s-backups/etcd/backup-$(date +%Y%m%d-%H%M%S).db
```

---

## 🔹 Interview Questions

### Architecture Questions

**Q1: Explain the Kubernetes architecture and main components.**
- **Control Plane**: API Server, etcd, Controller Manager, Scheduler
- **Worker Nodes**: kubelet, kube-proxy, Container Runtime
- **Add-ons**: DNS, Dashboard, Monitoring, Logging

**Q2: How does the Kubernetes scheduler work?**
- **Filtering**: Find nodes that meet pod requirements
- **Scoring**: Rank feasible nodes based on priorities
- **Selection**: Choose highest scoring node for pod placement

**Q3: What is the role of etcd in Kubernetes?**
- Distributed key-value store for cluster state
- Stores all configuration and resource definitions
- Provides strong consistency and high availability
- Only API server communicates with etcd

### Technical Questions

**Q4: How does service discovery work in Kubernetes?**
- DNS-based service resolution
- Environment variables for existing services
- Service endpoints automatically managed
- CoreDNS provides cluster DNS functionality

**Q5: Explain the difference between Docker and containerd.**
- **Docker**: Complete container platform with daemon
- **containerd**: Industry-standard container runtime
- **CRI**: Container Runtime Interface for Kubernetes
- **Migration**: Kubernetes moved from Docker to containerd

**Q6: How does Kubernetes networking work?**
- Every pod gets unique IP address
- Flat network namespace across cluster
- CNI plugins provide network implementation
- Services provide stable endpoints and load balancing

### Scenario-Based Questions

**Q7: Design a highly available Kubernetes cluster.**
```yaml
# Multi-master setup with load balancer
# External etcd cluster for better performance
# Multi-zone deployment for fault tolerance
# Proper backup and disaster recovery procedures
```

**Q8: How would you secure a Kubernetes cluster?**
```yaml
# Enable RBAC and Pod Security Standards
# Implement network policies
# Use service mesh for mTLS
# Regular security updates and scanning
# Audit logging and monitoring
```

**Q9: Troubleshoot a cluster where pods are not starting.**
```bash
# Check cluster component health
# Verify node resources and status
# Check scheduler and controller manager logs
# Validate network connectivity and DNS
# Review resource quotas and limits
```

### Best Practices Questions

**Q10: What are Kubernetes production best practices?**
- Implement proper resource management and quotas
- Use namespaces for multi-tenancy
- Enable comprehensive monitoring and logging
- Implement proper backup and disaster recovery
- Follow security best practices and regular updates
- Use Infrastructure as Code for cluster management
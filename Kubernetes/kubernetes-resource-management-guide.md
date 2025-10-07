# 💾 Kubernetes Resource Management - DevOps Interview Guide

## Table of Contents
1. [Resource Management Overview](#resource-management-overview)
2. [Resource Requests and Limits](#resource-requests-and-limits)
3. [Quality of Service (QoS)](#quality-of-service-qos)
4. [Resource Quotas](#resource-quotas)
5. [Limit Ranges](#limit-ranges)
6. [Node Resources](#node-resources)
7. [Resource Monitoring](#resource-monitoring)
8. [Practical Examples](#practical-examples)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Interview Questions](#interview-questions)

---

## 🔹 Resource Management Overview

### What is Resource Management?

**Resource Management** in Kubernetes controls how compute resources (CPU, memory, storage) are allocated, consumed, and limited across pods and containers.

### Resource Types

```yaml
# Compute Resources:
- CPU (measured in cores or millicores)
- Memory (measured in bytes)
- Ephemeral Storage (measured in bytes)
- Extended Resources (GPUs, custom resources)

# Resource Units:
CPU: "1" = 1 core, "500m" = 0.5 cores, "100m" = 0.1 cores
Memory: "1Gi" = 1 GiB, "512Mi" = 512 MiB, "1G" = 1 GB
Storage: "10Gi" = 10 GiB, "500Mi" = 500 MiB
```

### Resource Management Components

```yaml
# Key Components:
1. Resource Requests - Minimum guaranteed resources
2. Resource Limits - Maximum allowed resources
3. Resource Quotas - Namespace-level resource constraints
4. Limit Ranges - Default and boundary values
5. QoS Classes - Priority and eviction behavior
```

---

## 🔹 Resource Requests and Limits

### Basic Resource Specification

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:1.21
    resources:
      requests:
        cpu: "250m"        # 0.25 CPU cores
        memory: "256Mi"    # 256 MiB memory
        ephemeral-storage: "1Gi"  # 1 GiB storage
      limits:
        cpu: "500m"        # 0.5 CPU cores max
        memory: "512Mi"    # 512 MiB memory max
        ephemeral-storage: "2Gi"  # 2 GiB storage max
```

### Multi-Container Resource Management

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-resources
spec:
  containers:
  # Main application container
  - name: web-app
    image: nginx:1.21
    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
  
  # Sidecar logging container
  - name: log-collector
    image: fluentd:v1.14
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
  
  # Init container
  initContainers:
  - name: setup
    image: busybox:1.35
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
```

### Extended Resources (GPUs)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: gpu-app
    image: tensorflow/tensorflow:latest-gpu
    resources:
      requests:
        cpu: "1000m"
        memory: "2Gi"
        nvidia.com/gpu: 1      # Request 1 GPU
      limits:
        cpu: "2000m"
        memory: "4Gi"
        nvidia.com/gpu: 1      # Limit to 1 GPU
```

---

## 🔹 Quality of Service (QoS)

### QoS Classes

#### 1. Guaranteed QoS (Highest Priority)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "500m"        # Same as requests
        memory: "1Gi"      # Same as requests
```

#### 2. Burstable QoS (Medium Priority)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
      limits:
        cpu: "1000m"       # Higher than requests
        memory: "1Gi"      # Higher than requests
```

#### 3. BestEffort QoS (Lowest Priority)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    # No resources specified
```

### QoS Impact on Scheduling and Eviction

```yaml
# Eviction Priority (when node runs out of resources):
1. BestEffort pods (evicted first)
2. Burstable pods exceeding requests
3. Burstable pods within requests
4. Guaranteed pods (evicted last)

# Scheduling Priority:
- Guaranteed pods get priority placement
- Resource requests are used for scheduling decisions
- Limits don't affect scheduling
```

---

## 🔹 Resource Quotas

### Basic Resource Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    # Compute resources
    requests.cpu: "10"           # Total CPU requests
    requests.memory: "20Gi"      # Total memory requests
    limits.cpu: "20"             # Total CPU limits
    limits.memory: "40Gi"        # Total memory limits
    
    # Object counts
    pods: "50"                   # Maximum pods
    services: "10"               # Maximum services
    persistentvolumeclaims: "20" # Maximum PVCs
    secrets: "10"                # Maximum secrets
    configmaps: "10"             # Maximum ConfigMaps
```

### Advanced Resource Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: advanced-quota
  namespace: production
spec:
  hard:
    # Storage resources
    requests.storage: "100Gi"
    persistentvolumeclaims: "10"
    
    # Specific storage classes
    gold.storageclass.storage.k8s.io/requests.storage: "50Gi"
    silver.storageclass.storage.k8s.io/requests.storage: "100Gi"
    
    # Extended resources
    requests.nvidia.com/gpu: "4"
    
    # Object counts by type
    count/deployments.apps: "20"
    count/services: "15"
    count/ingresses.networking.k8s.io: "5"
  
  # Scope selectors
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority", "medium-priority"]
```

### Priority-Based Resource Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: priority-quota
  namespace: production
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: best-effort-quota
  namespace: production
spec:
  hard:
    pods: "10"
  scopes: ["BestEffort"]
```

---

## 🔹 Limit Ranges

### Basic Limit Range

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: development
spec:
  limits:
  # Container limits
  - type: Container
    default:                    # Default limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:             # Default requests
      cpu: "100m"
      memory: "128Mi"
    max:                        # Maximum allowed
      cpu: "2000m"
      memory: "4Gi"
    min:                        # Minimum required
      cpu: "50m"
      memory: "64Mi"
    maxLimitRequestRatio:       # Limit/Request ratio
      cpu: "4"                  # Limit can be 4x request
      memory: "2"               # Limit can be 2x request
```

### Advanced Limit Range

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: comprehensive-limits
  namespace: production
spec:
  limits:
  # Container constraints
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
      ephemeral-storage: "1Gi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
      ephemeral-storage: "500Mi"
    max:
      cpu: "4000m"
      memory: "8Gi"
      ephemeral-storage: "10Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
      ephemeral-storage: "100Mi"
  
  # Pod constraints
  - type: Pod
    max:
      cpu: "8000m"
      memory: "16Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
  
  # PVC constraints
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"
    min:
      storage: "1Gi"
```

---

## 🔹 Node Resources

### Node Capacity and Allocatable

```yaml
# Node resource information
apiVersion: v1
kind: Node
metadata:
  name: worker-node-1
status:
  capacity:
    cpu: "4"                    # Total CPU cores
    memory: "8Gi"               # Total memory
    ephemeral-storage: "100Gi"  # Total storage
    pods: "110"                 # Maximum pods
  allocatable:
    cpu: "3800m"                # Available for pods
    memory: "7.5Gi"             # Available for pods
    ephemeral-storage: "90Gi"   # Available for pods
    pods: "110"                 # Available pod slots
```

### Node Resource Reservation

```yaml
# Kubelet configuration for resource reservation
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
systemReserved:
  cpu: "100m"
  memory: "256Mi"
  ephemeral-storage: "5Gi"
kubeReserved:
  cpu: "100m"
  memory: "256Mi"
  ephemeral-storage: "5Gi"
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
```

---

## 🔹 Resource Monitoring

### Metrics Server

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server/metrics-server:v0.6.1
        args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        resources:
          requests:
            cpu: "100m"
            memory: "200Mi"
          limits:
            cpu: "200m"
            memory: "400Mi"
```

### Resource Monitoring Commands

```bash
# Check node resources
kubectl top nodes

# Check pod resources
kubectl top pods
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Describe node resources
kubectl describe node <node-name>

# Check resource quotas
kubectl describe quota -n <namespace>

# Check limit ranges
kubectl describe limitrange -n <namespace>
```

---

## 🔹 Practical Examples

### 1. Web Application with Resource Management

```yaml
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
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        
      - name: sidecar
        image: fluentd:v1.14
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
---
# Namespace Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: web-app-quota
  namespace: production
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
    pods: "10"
---
# Limit Range
apiVersion: v1
kind: LimitRange
metadata:
  name: web-app-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "1000m"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
```

### 2. Database with Guaranteed QoS

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        resources:
          requests:
            cpu: "1000m"      # Guaranteed QoS
            memory: "2Gi"     # requests = limits
          limits:
            cpu: "1000m"
            memory: "2Gi"
        env:
        - name: POSTGRES_DB
          value: "myapp"
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: "50Gi"
      storageClassName: "fast-ssd"
```

### 3. Batch Job with Resource Constraints

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  parallelism: 5
  completions: 10
  template:
    spec:
      containers:
      - name: processor
        image: data-processor:v1.0
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"      # Burstable QoS
            memory: "4Gi"     # Can burst for processing
        env:
        - name: BATCH_SIZE
          value: "1000"
      restartPolicy: Never
---
# Job-specific Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: batch-quota
  namespace: batch-processing
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
```

### 4. Multi-Tenant Namespace Setup

```yaml
# Development Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: development
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    services: "5"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "1000m"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
---
# Production Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
    services: "20"
    persistentvolumeclaims: "50"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    max:
      cpu: "4000m"
      memory: "8Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
```

---

## 🔹 Best Practices

### 1. Resource Specification

```yaml
# Always set resource requests
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# Guidelines:
- Set requests based on actual usage
- Set limits to prevent resource hogging
- Use Guaranteed QoS for critical workloads
- Monitor and adjust based on metrics
```

### 2. QoS Class Selection

```yaml
# Critical services (databases, core APIs)
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"      # Guaranteed QoS
    memory: "2Gi"

# Regular applications
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"       # Burstable QoS
    memory: "512Mi"

# Batch jobs (non-critical)
# No resources specified = BestEffort QoS
```

### 3. Namespace Resource Management

```yaml
# Implement resource quotas per namespace
# Set appropriate limit ranges
# Monitor resource usage regularly
# Adjust quotas based on actual needs
```

### 4. Monitoring and Alerting

```yaml
# Set up alerts for:
- High resource utilization (>80%)
- Resource quota exhaustion
- Pod evictions due to resource pressure
- Nodes running out of resources
```

---

## 🔹 Troubleshooting

### Common Resource Issues

#### 1. Pod Stuck in Pending (Insufficient Resources)

```bash
# Check pod events
kubectl describe pod <pod-name>

# Check node resources
kubectl top nodes
kubectl describe nodes

# Common causes:
# - Insufficient CPU/memory on nodes
# - Resource requests too high
# - Resource quotas exceeded
```

#### 2. Pod Evicted (Resource Pressure)

```bash
# Check eviction events
kubectl get events --field-selector reason=Evicted

# Check node conditions
kubectl describe node <node-name> | grep Conditions

# Common causes:
# - Node memory pressure
# - Node disk pressure
# - BestEffort pods evicted first
```

#### 3. Resource Quota Exceeded

```bash
# Check quota status
kubectl describe quota -n <namespace>

# Check current usage
kubectl top pods -n <namespace>

# Solutions:
# - Increase quota limits
# - Reduce resource requests
# - Delete unused resources
```

### Debugging Commands

```bash
# Resource monitoring
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Resource quotas
kubectl get quota -A
kubectl describe quota <quota-name> -n <namespace>

# Limit ranges
kubectl get limitrange -A
kubectl describe limitrange <lr-name> -n <namespace>

# Node resource allocation
kubectl describe node <node-name>

# Pod resource usage
kubectl describe pod <pod-name>
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: Explain the difference between resource requests and limits.**
- **Requests**: Minimum guaranteed resources for scheduling
- **Limits**: Maximum resources a container can use

**Q2: What are the QoS classes in Kubernetes?**
- **Guaranteed**: requests = limits (highest priority)
- **Burstable**: requests < limits (medium priority)
- **BestEffort**: no requests/limits (lowest priority)

**Q3: How does Kubernetes handle resource contention?**
- Uses QoS classes for eviction priority
- Evicts BestEffort pods first
- Then Burstable pods exceeding requests
- Guaranteed pods evicted last

### Technical Questions

**Q4: What happens when a pod exceeds its memory limit?**
- Pod gets OOMKilled (Out of Memory)
- Container restarts based on restart policy
- Event logged in cluster

**Q5: How do resource quotas work?**
- Namespace-level resource constraints
- Prevent resource overconsumption
- Block pod creation if quota exceeded
- Track both requests and limits

**Q6: What is the difference between capacity and allocatable?**
- **Capacity**: Total node resources
- **Allocatable**: Resources available for pods (capacity - reserved)

### Scenario-Based Questions

**Q7: Design resource management for a multi-tenant cluster.**
```yaml
# Separate namespaces per tenant
# Resource quotas per namespace
# Limit ranges for defaults
# Priority classes for critical workloads
# Monitoring and alerting
```

**Q8: How would you handle a memory leak in production?**
```bash
# Monitor resource usage trends
# Set appropriate memory limits
# Implement health checks
# Use circuit breakers
# Plan for graceful degradation
```

**Q9: Troubleshoot pods being evicted frequently.**
```bash
# Check node resource pressure
# Review QoS classes
# Adjust resource requests/limits
# Monitor application memory usage
# Consider node scaling
```

### Best Practices Questions

**Q10: What are resource management best practices?**
- Always set resource requests and limits
- Use appropriate QoS classes
- Implement resource quotas and limit ranges
- Monitor resource usage continuously
- Plan for peak load scenarios
- Regular capacity planning
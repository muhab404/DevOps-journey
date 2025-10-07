# 🎯 Kubernetes Scheduling & Advanced Scheduling - DevOps Interview Guide

## Table of Contents
1. [Scheduling Overview](#scheduling-overview)
2. [Basic Scheduling](#basic-scheduling)
3. [Node Selection](#node-selection)
4. [Affinity and Anti-Affinity](#affinity-and-anti-affinity)
5. [Taints and Tolerations](#taints-and-tolerations)
6. [Advanced Scheduling](#advanced-scheduling)
7. [Custom Schedulers](#custom-schedulers)
8. [Practical Examples](#practical-examples)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Interview Questions](#interview-questions)

---

## 🔹 Scheduling Overview

### What is Kubernetes Scheduling?

**Scheduling** is the process of assigning pods to nodes in a Kubernetes cluster based on resource requirements, constraints, and policies.

### Scheduling Process

```yaml
# Scheduling Flow:
API Server → Scheduler → Node Selection → Kubelet → Pod Creation

# Scheduler Decision Process:
1. Filtering (Predicates) - Find feasible nodes
2. Scoring (Priorities) - Rank feasible nodes
3. Selection - Choose highest scoring node
```

### Default Scheduler

```yaml
# kube-scheduler responsibilities:
- Resource availability checking
- Node affinity evaluation
- Pod affinity/anti-affinity
- Taints and tolerations
- Quality of Service (QoS) considerations
```

---

## 🔹 Basic Scheduling

### Resource-Based Scheduling

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    resources:
      requests:
        cpu: "500m"      # 0.5 CPU cores
        memory: "1Gi"    # 1 GiB memory
      limits:
        cpu: "1000m"     # 1 CPU core
        memory: "2Gi"    # 2 GiB memory
```

### Quality of Service (QoS) Classes

```yaml
# 1. Guaranteed QoS (Highest Priority)
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "500m"      # Same as requests
    memory: "1Gi"    # Same as requests

# 2. Burstable QoS (Medium Priority)
resources:
  requests:
    cpu: "200m"
    memory: "512Mi"
  limits:
    cpu: "1000m"     # Higher than requests
    memory: "2Gi"    # Higher than requests

# 3. BestEffort QoS (Lowest Priority)
# No resources specified
```

---

## 🔹 Node Selection

### Node Selector (Simple)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector-pod
spec:
  nodeSelector:
    disktype: ssd           # Node must have this label
    zone: us-west-1a        # Node must be in this zone
    instance-type: c5.large # Node must be this instance type
  containers:
  - name: app
    image: nginx:1.21
```

### Node Name (Direct Assignment)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-name-pod
spec:
  nodeName: worker-node-1   # Schedule directly to this node
  containers:
  - name: app
    image: nginx:1.21
```

---

## 🔹 Affinity and Anti-Affinity

### Node Affinity

#### Required Node Affinity (Hard Constraint)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          # Node must have SSD disk
          - key: disktype
            operator: In
            values: ["ssd"]
          # Node must NOT be spot instance
          - key: instance-lifecycle
            operator: NotIn
            values: ["spot"]
          # Node must exist in specific zones
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-west-1a", "us-west-1b"]
  containers:
  - name: app
    image: nginx:1.21
```

#### Preferred Node Affinity (Soft Constraint)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      # Prefer nodes with high memory (weight 80)
      - weight: 80
        preference:
          matchExpressions:
          - key: node.kubernetes.io/instance-type
            operator: In
            values: ["r5.large", "r5.xlarge"]
      # Prefer nodes in specific zone (weight 20)
      - weight: 20
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-west-1a"]
  containers:
  - name: app
    image: nginx:1.21
```

### Pod Affinity

#### Pod Affinity (Schedule Together)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["database"]
        topologyKey: kubernetes.io/hostname  # Same node
      
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: cache
          topologyKey: topology.kubernetes.io/zone  # Same zone
  containers:
  - name: app
    image: nginx:1.21
```

#### Pod Anti-Affinity (Schedule Apart)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-anti-affinity-pod
  labels:
    app: web-server
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web-server
        topologyKey: kubernetes.io/hostname  # Different nodes
      
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web-server
          topologyKey: topology.kubernetes.io/zone  # Different zones
  containers:
  - name: app
    image: nginx:1.21
```

### Topology Spread Constraints

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: topology-spread-pod
  labels:
    app: web-app
spec:
  topologySpreadConstraints:
  # Spread across zones
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web-app
  # Spread across nodes
  - maxSkew: 2
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web-app
  containers:
  - name: app
    image: nginx:1.21
```

---

## 🔹 Taints and Tolerations

### Node Taints

```bash
# Add taint to node
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key1=value1:PreferNoSchedule

# Remove taint
kubectl taint nodes node1 key1=value1:NoSchedule-

# View node taints
kubectl describe node node1 | grep Taints
```

### Taint Effects

```yaml
# NoSchedule: Pods won't be scheduled unless they tolerate
# NoExecute: Existing pods will be evicted unless they tolerate
# PreferNoSchedule: Soft version of NoSchedule
```

### Pod Tolerations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  tolerations:
  # Exact match toleration
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  
  # Exists toleration (any value)
  - key: "special-hardware"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300  # Tolerate for 5 minutes
  
  # Tolerate all taints on master nodes
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
  
  # Tolerate any taint
  - operator: "Exists"
  
  containers:
  - name: app
    image: nginx:1.21
```

---

## 🔹 Advanced Scheduling

### Priority Classes

```yaml
# Define Priority Class
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority class for critical workloads"
---
# Use Priority Class in Pod
apiVersion: v1
kind: Pod
metadata:
  name: priority-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx:1.21
```

### Preemption

```yaml
# High priority pod can preempt lower priority pods
apiVersion: v1
kind: Pod
metadata:
  name: preempting-pod
spec:
  priorityClassName: high-priority
  preemptionPolicy: PreemptLowerPriority  # Default behavior
  containers:
  - name: app
    image: nginx:1.21
    resources:
      requests:
        cpu: "2000m"
        memory: "4Gi"
```

### Pod Disruption Budgets (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2        # Minimum pods available
  # maxUnavailable: 1    # Alternative: maximum unavailable
  selector:
    matchLabels:
      app: web-app
```

### Resource Quotas and Limits

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: production
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2000m"
      memory: "4Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    type: Container
```

---

## 🔹 Custom Schedulers

### Custom Scheduler Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-config
  namespace: kube-system
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1beta3
    kind: KubeSchedulerConfiguration
    profiles:
    - schedulerName: my-scheduler
      plugins:
        score:
          enabled:
          - name: NodeResourcesFit
          - name: NodeAffinity
          disabled:
          - name: NodeResourcesLeastAllocated
      pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: LeastAllocated
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-scheduler
  template:
    metadata:
      labels:
        app: my-scheduler
    spec:
      containers:
      - name: kube-scheduler
        image: k8s.gcr.io/kube-scheduler:v1.28.0
        command:
        - kube-scheduler
        - --config=/etc/kubernetes/my-scheduler-config.yaml
        - --v=2
        volumeMounts:
        - name: config-volume
          mountPath: /etc/kubernetes
      volumes:
      - name: config-volume
        configMap:
          name: my-scheduler-config
```

### Using Custom Scheduler

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduled-pod
spec:
  schedulerName: my-scheduler  # Use custom scheduler
  containers:
  - name: app
    image: nginx:1.21
```

---

## 🔹 Practical Examples

### 1. High Availability Web Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-web-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: ha-web-app
  template:
    metadata:
      labels:
        app: ha-web-app
    spec:
      # Spread across zones and nodes
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ha-web-app
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: ha-web-app
      
      # Anti-affinity for better distribution
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: ha-web-app
              topologyKey: kubernetes.io/hostname
      
      # Prefer nodes with SSD
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
      
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
---
# Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ha-web-app-pdb
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: ha-web-app
```

### 2. Database with Dedicated Nodes

```yaml
# Taint dedicated database nodes
# kubectl taint nodes db-node-1 dedicated=database:NoSchedule
# kubectl taint nodes db-node-2 dedicated=database:NoSchedule

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 2
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      # Tolerate database node taints
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "database"
        effect: "NoSchedule"
      
      # Require database nodes
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values: ["database"]
        
        # Anti-affinity for HA
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: database
            topologyKey: kubernetes.io/hostname
      
      # High priority
      priorityClassName: high-priority
      
      containers:
      - name: postgres
        image: postgres:13
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
```

### 3. GPU Workload Scheduling

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  # Require GPU nodes
  nodeSelector:
    accelerator: nvidia-tesla-k80
  
  # Tolerate GPU node taints
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"
  
  # High priority for expensive resources
  priorityClassName: gpu-priority
  
  containers:
  - name: gpu-app
    image: tensorflow/tensorflow:latest-gpu
    resources:
      requests:
        nvidia.com/gpu: 1
        cpu: "2000m"
        memory: "4Gi"
      limits:
        nvidia.com/gpu: 1
        cpu: "4000m"
        memory: "8Gi"
```

### 4. Multi-Tier Application with Affinity

```yaml
# Frontend Deployment
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
      # Prefer to be close to backend
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  tier: backend
              topologyKey: topology.kubernetes.io/zone
        
        # Spread across nodes
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: frontend
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: frontend
        image: nginx:1.21
---
# Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      # Must be close to database
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                tier: database
            topologyKey: topology.kubernetes.io/zone
        
        # Spread across nodes in same zone
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: backend
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: backend
        image: backend:v1.0
```

---

## 🔹 Best Practices

### 1. Resource Management

```yaml
# Always set resource requests
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# Use appropriate QoS classes
# Guaranteed for critical workloads
# Burstable for most applications
# BestEffort only for non-critical batch jobs
```

### 2. High Availability

```yaml
# Use pod anti-affinity for HA
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          app: my-app
      topologyKey: kubernetes.io/hostname

# Use topology spread constraints
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
```

### 3. Performance Optimization

```yaml
# Co-locate related services
podAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          app: database
      topologyKey: kubernetes.io/hostname
```

### 4. Security and Isolation

```yaml
# Use dedicated nodes for sensitive workloads
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "secure"
  effect: "NoSchedule"

nodeSelector:
  security-level: high
```

---

## 🔹 Troubleshooting

### Common Scheduling Issues

#### 1. Pod Stuck in Pending State

```bash
# Check pod events
kubectl describe pod <pod-name>

# Common causes:
# - Insufficient resources
# - Node selector/affinity not matching
# - Taints without tolerations
# - PodDisruptionBudget blocking
```

#### 2. Unschedulable Nodes

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check node capacity
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
```

#### 3. Affinity/Anti-Affinity Issues

```bash
# Check pod labels and selectors
kubectl get pods --show-labels
kubectl describe pod <pod-name> | grep -A 10 "Affinity"
```

### Debugging Commands

```bash
# Check scheduler logs
kubectl logs -n kube-system -l component=kube-scheduler

# Check node resources
kubectl top nodes
kubectl describe nodes

# Check pod scheduling
kubectl get events --sort-by=.metadata.creationTimestamp

# Check taints and tolerations
kubectl describe node <node-name> | grep Taints
kubectl describe pod <pod-name> | grep -A 5 Tolerations

# Check priority classes
kubectl get priorityclasses
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: Explain the Kubernetes scheduling process.**
1. **Filtering**: Find nodes that meet pod requirements
2. **Scoring**: Rank feasible nodes based on priorities
3. **Selection**: Choose highest scoring node

**Q2: What's the difference between nodeSelector and nodeAffinity?**
- **nodeSelector**: Simple key-value matching (legacy)
- **nodeAffinity**: Advanced expressions with required/preferred rules

**Q3: Explain pod affinity vs anti-affinity.**
- **Affinity**: Schedule pods together (same node/zone)
- **Anti-affinity**: Schedule pods apart (different nodes/zones)

### Technical Questions

**Q4: How do taints and tolerations work?**
- **Taints**: Applied to nodes to repel pods
- **Tolerations**: Applied to pods to tolerate taints
- Effects: NoSchedule, PreferNoSchedule, NoExecute

**Q5: What are the QoS classes in Kubernetes?**
- **Guaranteed**: requests = limits (highest priority)
- **Burstable**: requests < limits (medium priority)
- **BestEffort**: no requests/limits (lowest priority)

**Q6: How does pod preemption work?**
- Higher priority pods can evict lower priority pods
- Scheduler finds nodes by preempting lower priority pods
- Graceful termination with terminationGracePeriodSeconds

### Scenario-Based Questions

**Q7: Design scheduling for a multi-tier application.**
```yaml
# Frontend: Spread across zones, prefer edge nodes
# Backend: Co-locate with cache, spread across nodes
# Database: Dedicated nodes, anti-affinity for HA
# Cache: Co-locate with backend, high memory nodes
```

**Q8: How would you handle GPU workloads?**
```yaml
# Use node selectors for GPU nodes
# Add tolerations for GPU taints
# Set resource requests for GPUs
# Use priority classes for expensive resources
```

**Q9: Troubleshoot pods not scheduling.**
```bash
# Check pod events and status
# Verify node resources and capacity
# Check taints, tolerations, and affinity rules
# Validate resource requests vs node capacity
# Check PodDisruptionBudgets and quotas
```

### Best Practices Questions

**Q10: What are scheduling best practices?**
- Always set resource requests and limits
- Use anti-affinity for high availability
- Implement proper priority classes
- Use topology spread constraints
- Monitor and alert on scheduling failures
- Test scheduling policies under load
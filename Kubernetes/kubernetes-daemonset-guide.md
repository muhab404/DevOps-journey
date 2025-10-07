# 🔄 Kubernetes DaemonSet - DevOps Interview Guide

## Table of Contents
1. [DaemonSet Overview](#daemonset-overview)
2. [Core Concepts](#core-concepts)
3. [DaemonSet Manifest Attributes](#daemonset-manifest-attributes)
4. [Scheduling & Node Selection](#scheduling--node-selection)
5. [Update Strategies](#update-strategies)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Interview Questions](#interview-questions)

---

## 🔹 DaemonSet Overview

### What is a DaemonSet?

**DaemonSet** ensures that all (or some) nodes run a copy of a specific pod. As nodes are added to the cluster, pods are added to them. As nodes are removed, those pods are garbage collected.

**Key Characteristics:**
- One pod per node (typically)
- Automatic pod scheduling on new nodes
- Automatic cleanup when nodes are removed
- Bypasses default scheduler for system-level tasks

### When to Use DaemonSets

```yaml
# Common Use Cases:
- Log collection agents (Fluentd, Filebeat)
- Monitoring agents (Node Exporter, Datadog Agent)
- Network plugins (Calico, Flannel)
- Storage daemons (Ceph, GlusterFS)
- Security agents (Falco, Twistlock)
- Node maintenance tools
```

### DaemonSet vs Other Controllers

| Controller | Purpose | Pod Distribution |
|------------|---------|------------------|
| **DaemonSet** | System services | One per node |
| **Deployment** | Stateless apps | Distributed across nodes |
| **StatefulSet** | Stateful apps | Ordered, persistent identity |
| **Job** | Batch tasks | Run to completion |

---

## 🔹 Core Concepts

### Pod Scheduling Behavior

```yaml
# DaemonSet pods are scheduled differently:
1. Bypass default scheduler
2. Ignore node unschedulable status
3. Respect node selectors and affinity rules
4. Can run on master nodes (with tolerations)
```

### Node Selection Mechanisms

```yaml
# 1. Node Selector (Simple)
nodeSelector:
  disktype: ssd

# 2. Node Affinity (Advanced)
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]

# 3. Tolerations (Master nodes, taints)
tolerations:
- key: node-role.kubernetes.io/master
  operator: Exists
  effect: NoSchedule
```

---

## 🔹 DaemonSet Manifest Attributes

### Complete DaemonSet Configuration

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  namespace: kube-system
  labels:
    app: monitoring-agent
    component: node-exporter
    version: v1.0.0
  annotations:
    description: "Prometheus Node Exporter DaemonSet"
spec:
  # Pod Selection
  selector:
    matchLabels:
      app: monitoring-agent
      component: node-exporter
  
  # Update Strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1        # Max pods unavailable during update
      maxSurge: 0              # DaemonSet doesn't support maxSurge
  
  # Minimum ready seconds
  minReadySeconds: 10
  
  # Revision history
  revisionHistoryLimit: 10
  
  # Pod Template
  template:
    metadata:
      labels:
        app: monitoring-agent
        component: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
    spec:
      # Host Network Access
      hostNetwork: true
      hostPID: true
      
      # DNS Policy for host network
      dnsPolicy: ClusterFirstWithHostNet
      
      # Service Account
      serviceAccountName: monitoring-agent
      
      # Security Context
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      
      # Node Selection
      nodeSelector:
        kubernetes.io/os: linux
      
      # Tolerations for master nodes
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      
      # Affinity Rules
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values: ["amd64", "arm64"]
      
      # Init Containers
      initContainers:
      - name: setup
        image: busybox:1.35
        command: ['sh', '-c']
        args:
        - |
          echo "Setting up monitoring agent..."
          mkdir -p /host/var/lib/node-exporter
        volumeMounts:
        - name: host-root
          mountPath: /host
          readOnly: false
        securityContext:
          privileged: true
      
      # Main Containers
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
        - --web.listen-address=0.0.0.0:9100
        
        # Ports
        ports:
        - name: metrics
          containerPort: 9100
          hostPort: 9100
          protocol: TCP
        
        # Health Checks
        livenessProbe:
          httpGet:
            path: /metrics
            port: 9100
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /metrics
            port: 9100
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        # Resources
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        
        # Security Context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        # Volume Mounts
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
        - name: tmp
          mountPath: /tmp
      
      # Volumes
      volumes:
      - name: proc
        hostPath:
          path: /proc
          type: Directory
      - name: sys
        hostPath:
          path: /sys
          type: Directory
      - name: root
        hostPath:
          path: /
          type: Directory
      - name: tmp
        emptyDir: {}
      
      # Termination Grace Period
      terminationGracePeriodSeconds: 30
      
      # Priority Class
      priorityClassName: system-node-critical
```

### Key Spec Fields Explained

```yaml
# Update Strategy Options
updateStrategy:
  type: RollingUpdate    # Only option for DaemonSet
  rollingUpdate:
    maxUnavailable: 1    # Number or percentage of pods that can be unavailable

# Alternative: OnDelete (manual updates)
updateStrategy:
  type: OnDelete         # Pods updated only when manually deleted
```

---

## 🔹 Scheduling & Node Selection

### Node Selector Examples

```yaml
# 1. Simple Node Selection
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
        zone: us-west-1

# 2. Exclude specific nodes
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: NotIn
                values: ["spot-instance"]
```

### Tolerations for Special Nodes

```yaml
# Master/Control Plane Tolerations
tolerations:
- key: node-role.kubernetes.io/master
  operator: Exists
  effect: NoSchedule
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule

# Custom Taint Tolerations
tolerations:
- key: dedicated
  operator: Equal
  value: monitoring
  effect: NoSchedule
- key: gpu-node
  operator: Exists
  effect: NoExecute
```

### Advanced Node Affinity

```yaml
affinity:
  nodeAffinity:
    # Hard requirement
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: node.kubernetes.io/instance-type
          operator: NotIn
          values: ["t2.nano", "t2.micro"]
    
    # Soft preference
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values: ["us-west-1a", "us-west-1b"]
```

---

## 🔹 Update Strategies

### Rolling Update Configuration

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%    # Can be number or percentage
  
  minReadySeconds: 30        # Wait time before considering pod ready
```

### OnDelete Update Strategy

```yaml
spec:
  updateStrategy:
    type: OnDelete
  
# Manual update process:
# 1. Update DaemonSet spec
# 2. Delete pods manually: kubectl delete pod <pod-name>
# 3. New pods created with updated spec
```

### Update Process Monitoring

```bash
# Watch rollout status
kubectl rollout status daemonset/my-daemonset

# Check rollout history
kubectl rollout history daemonset/my-daemonset

# Rollback to previous version
kubectl rollout undo daemonset/my-daemonset
```

---

## 🔹 Practical Examples

### 1. Log Collection Agent (Fluentd)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch7-1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config-volume
          mountPath: /fluentd/etc
      
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-config
```

### 2. Network Plugin (Calico)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      - operator: Exists
        effect: NoExecute
      
      serviceAccountName: calico-node
      priorityClassName: system-node-critical
      
      initContainers:
      - name: upgrade-ipam
        image: calico/cni:v3.26.1
        command: ["/opt/cni/bin/calico-ipam", "-upgrade"]
        volumeMounts:
        - mountPath: /var/lib/cni/networks
          name: host-local-net-dir
        - mountPath: /host/opt/cni/bin
          name: cni-bin-dir
      
      containers:
      - name: calico-node
        image: calico/node:v3.26.1
        env:
        - name: DATASTORE_TYPE
          value: "kubernetes"
        - name: CALICO_NETWORKING_BACKEND
          value: "bird"
        - name: CLUSTER_TYPE
          value: "k8s,bgp"
        
        securityContext:
          privileged: true
        
        resources:
          requests:
            cpu: 250m
        
        volumeMounts:
        - mountPath: /lib/modules
          name: lib-modules
          readOnly: true
        - mountPath: /run/xtables.lock
          name: xtables-lock
        - mountPath: /var/run/calico
          name: var-run-calico
        - mountPath: /var/lib/calico
          name: var-lib-calico
      
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: var-run-calico
        hostPath:
          path: /var/run/calico
      - name: var-lib-calico
        hostPath:
          path: /var/lib/calico
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```

### 3. Security Agent (Falco)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: falco-system
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      serviceAccountName: falco
      hostNetwork: true
      hostPID: true
      
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      
      containers:
      - name: falco
        image: falcosecurity/falco:0.35.1
        args:
        - /usr/bin/falco
        - --cri=/run/containerd/containerd.sock
        - --k8s-api=https://kubernetes.default.svc.cluster.local
        - --k8s-api-cert=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        - --k8s-api-token-file=/var/run/secrets/kubernetes.io/serviceaccount/token
        
        securityContext:
          privileged: true
        
        resources:
          limits:
            memory: 1Gi
            cpu: 1000m
          requests:
            memory: 512Mi
            cpu: 100m
        
        volumeMounts:
        - mountPath: /host/var/run/docker.sock
          name: docker-socket
        - mountPath: /host/run/containerd/containerd.sock
          name: containerd-socket
        - mountPath: /host/dev
          name: dev-fs
        - mountPath: /host/proc
          name: proc-fs
          readOnly: true
        - mountPath: /host/boot
          name: boot-fs
          readOnly: true
        - mountPath: /host/lib/modules
          name: lib-modules
          readOnly: true
        - mountPath: /host/usr
          name: usr-fs
          readOnly: true
        - mountPath: /host/etc
          name: etc-fs
          readOnly: true
      
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
      - name: containerd-socket
        hostPath:
          path: /run/containerd/containerd.sock
      - name: dev-fs
        hostPath:
          path: /dev
      - name: proc-fs
        hostPath:
          path: /proc
      - name: boot-fs
        hostPath:
          path: /boot
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: usr-fs
        hostPath:
          path: /usr
      - name: etc-fs
        hostPath:
          path: /etc
```

---

## 🔹 Best Practices

### 1. Resource Management

```yaml
# Always set resource requests and limits
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "200m"

# Use appropriate QoS class
# Guaranteed: requests = limits (recommended for system pods)
```

### 2. Security Hardening

```yaml
# Minimal security context
securityContext:
  runAsNonRoot: true
  runAsUser: 65534
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # Only if needed

# Use dedicated service account
serviceAccountName: daemonset-sa
```

### 3. Health Checks

```yaml
# Comprehensive health checks
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3
```

### 4. Update Strategy

```yaml
# Safe rolling update
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1    # Conservative approach

minReadySeconds: 30      # Allow time for stabilization
```

### 5. Monitoring and Observability

```yaml
# Add monitoring annotations
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9100"
    prometheus.io/path: "/metrics"

# Use meaningful labels
labels:
  app: monitoring-agent
  component: node-exporter
  version: v1.6.1
  tier: infrastructure
```

---

## 🔹 Troubleshooting

### Common Issues and Solutions

#### 1. Pods Not Scheduled on All Nodes

```bash
# Check node taints
kubectl describe nodes | grep -A 5 Taints

# Check node selectors
kubectl get daemonset my-ds -o yaml | grep -A 10 nodeSelector

# Solution: Add appropriate tolerations
tolerations:
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 300
```

#### 2. Update Stuck in Progress

```bash
# Check rollout status
kubectl rollout status daemonset/my-ds

# Check pod events
kubectl describe pods -l app=my-ds

# Force restart if needed
kubectl rollout restart daemonset/my-ds
```

#### 3. Resource Constraints

```bash
# Check resource usage
kubectl top pods -l app=my-ds

# Check node resources
kubectl describe nodes

# Solution: Adjust resource requests/limits
resources:
  requests:
    memory: "32Mi"    # Reduce if too high
    cpu: "25m"
```

### Debugging Commands

```bash
# List DaemonSets
kubectl get daemonsets -A

# Describe DaemonSet
kubectl describe daemonset <name> -n <namespace>

# Check pod distribution
kubectl get pods -o wide -l app=<daemonset-label>

# View logs
kubectl logs daemonset/<name> -n <namespace>

# Check rollout history
kubectl rollout history daemonset/<name>

# Manual rollback
kubectl rollout undo daemonset/<name>

# Delete and recreate pods
kubectl delete pods -l app=<daemonset-label>
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: What is a DaemonSet and when would you use it?**
- Ensures one pod per node
- Used for system-level services (logging, monitoring, networking)
- Automatically handles node additions/removals

**Q2: How does DaemonSet scheduling differ from Deployment?**
- DaemonSet bypasses default scheduler
- Ignores node unschedulable status
- Respects node selectors and tolerations
- One pod per node (not distributed)

**Q3: Explain DaemonSet update strategies.**
- **RollingUpdate**: Updates pods gradually (default)
- **OnDelete**: Manual updates by deleting pods
- maxUnavailable controls update pace

### Technical Questions

**Q4: How do you ensure DaemonSet pods run on master nodes?**
```yaml
tolerations:
- key: node-role.kubernetes.io/master
  operator: Exists
  effect: NoSchedule
```

**Q5: What happens when a new node joins the cluster?**
- DaemonSet controller detects new node
- Automatically schedules pod on new node
- Pod starts with current DaemonSet spec

**Q6: How do you exclude certain nodes from DaemonSet?**
```yaml
# Use node affinity with NotIn operator
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-type
          operator: NotIn
          values: ["spot-instance"]
```

### Scenario-Based Questions

**Q7: Design a monitoring solution using DaemonSet.**
```yaml
# Node Exporter for metrics collection
# Fluentd for log aggregation  
# Both need host access and run on all nodes
# Use tolerations for master nodes
# Set resource limits to prevent resource starvation
```

**Q8: Troubleshoot a DaemonSet where pods aren't starting on some nodes.**
```bash
# Check node taints and tolerations
kubectl describe nodes
kubectl get ds <name> -o yaml | grep -A 10 tolerations

# Check resource availability
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check node selectors
kubectl get ds <name> -o yaml | grep -A 5 nodeSelector
```

**Q9: How do you perform a safe DaemonSet update?**
```yaml
# Use RollingUpdate with conservative settings
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1

# Set minReadySeconds for stability
minReadySeconds: 30

# Monitor rollout
kubectl rollout status daemonset/<name>
```

### Best Practices Questions

**Q10: Security considerations for DaemonSets?**
- Use dedicated ServiceAccounts with minimal permissions
- Run as non-root user when possible
- Use read-only root filesystem
- Drop unnecessary capabilities
- Regular security scanning of images
- Network policies for pod communication
# 📈 Kubernetes Autoscaling - DevOps Interview Guide

## Table of Contents
1. [Autoscaling Overview](#autoscaling-overview)
2. [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa)
3. [Vertical Pod Autoscaler (VPA)](#vertical-pod-autoscaler-vpa)
4. [Cluster Autoscaler](#cluster-autoscaler)
5. [Custom Metrics & External Metrics](#custom-metrics--external-metrics)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Interview Questions](#interview-questions)

---

## 🔹 Autoscaling Overview

### Types of Autoscaling

**Kubernetes provides three main autoscaling mechanisms:**

| Type | Scope | What it scales | Trigger |
|------|-------|----------------|---------|
| **HPA** | Pods | Number of replicas | CPU, Memory, Custom metrics |
| **VPA** | Pods | Resource requests/limits | Resource utilization |
| **CA** | Nodes | Number of nodes | Pod scheduling failures |

### Autoscaling Architecture

```yaml
# Autoscaling Flow:
Metrics Server → HPA Controller → Deployment → Pods
                ↓
Custom Metrics API → External Metrics → HPA
                ↓
Cluster Autoscaler → Node Pool → New Nodes
```

### Prerequisites

```yaml
# Required Components:
1. Metrics Server (for CPU/Memory metrics)
2. Custom Metrics API (for custom metrics)
3. External Metrics API (for external metrics)
4. Proper resource requests/limits on pods
```

---

## 🔹 Horizontal Pod Autoscaler (HPA)

### HPA Basics

**HPA** automatically scales the number of pods based on observed metrics (CPU, memory, or custom metrics).

### Basic HPA Manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
  labels:
    app: web-app
    component: autoscaler
spec:
  # Target deployment
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  
  # Scaling bounds
  minReplicas: 2
  maxReplicas: 50
  
  # Scaling metrics
  metrics:
  # CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Scaling behavior
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      selectPolicy: Min
```

### HPA Metric Types

#### 1. Resource Metrics (CPU/Memory)

```yaml
metrics:
# CPU percentage
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70

# Memory percentage
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80

# Absolute CPU value
- type: Resource
  resource:
    name: cpu
    target:
      type: AverageValue
      averageValue: "500m"
```

#### 2. Pod Metrics

```yaml
metrics:
# Custom pod metric
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"

# Pod metric with selector
- type: Pods
  pods:
    metric:
      name: queue_length
      selector:
        matchLabels:
          queue: "worker"
    target:
      type: AverageValue
      averageValue: "30"
```

#### 3. Object Metrics

```yaml
metrics:
# Service-based metric
- type: Object
  object:
    metric:
      name: requests_per_second
    describedObject:
      apiVersion: v1
      kind: Service
      name: web-service
    target:
      type: Value
      value: "2000"

# Ingress-based metric
- type: Object
  object:
    metric:
      name: ingress_requests_per_second
    describedObject:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: web-ingress
    target:
      type: AverageValue
      averageValue: "500"
```

#### 4. External Metrics

```yaml
metrics:
# SQS queue length
- type: External
  external:
    metric:
      name: sqs_queue_length
      selector:
        matchLabels:
          queue: "worker-queue"
    target:
      type: Value
      value: "100"

# Prometheus metric
- type: External
  external:
    metric:
      name: prometheus_metric
      selector:
        matchLabels:
          job: "web-app"
    target:
      type: AverageValue
      averageValue: "50"
```

### HPA Scaling Behavior

```yaml
behavior:
  scaleUp:
    # Stabilization window (prevent flapping)
    stabilizationWindowSeconds: 60
    
    # Scaling policies
    policies:
    - type: Percent      # Scale by percentage
      value: 100         # Double the pods
      periodSeconds: 15  # Every 15 seconds
    - type: Pods         # Scale by absolute number
      value: 4           # Add 4 pods
      periodSeconds: 15
    
    # Policy selection
    selectPolicy: Max    # Use the policy that scales more
  
  scaleDown:
    stabilizationWindowSeconds: 300  # 5 minutes
    policies:
    - type: Percent
      value: 10          # Scale down by 10%
      periodSeconds: 60  # Every minute
    selectPolicy: Min    # Use the policy that scales less
```

---

## 🔹 Vertical Pod Autoscaler (VPA)

### VPA Overview

**VPA** automatically adjusts CPU and memory requests/limits for containers based on usage patterns.

### VPA Modes

```yaml
# Update Modes:
- "Off"        # Only provide recommendations
- "Initial"    # Set resources on pod creation only
- "Auto"       # Automatically update running pods (restarts pods)
```

### Basic VPA Manifest

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
  namespace: production
spec:
  # Target deployment
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  
  # Update policy
  updatePolicy:
    updateMode: "Auto"        # Off, Initial, Auto
    minReplicas: 2            # Minimum replicas during updates
  
  # Resource policy
  resourcePolicy:
    containerPolicies:
    - containerName: web-container
      # Resource bounds
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2000m"
        memory: "4Gi"
      
      # Controlled resources
      controlledResources: ["cpu", "memory"]
      
      # Controlled values
      controlledValues: RequestsAndLimits  # RequestsOnly, RequestsAndLimits
```

### Advanced VPA Configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: advanced-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: multi-container-app
  
  updatePolicy:
    updateMode: "Auto"
    minReplicas: 1
  
  resourcePolicy:
    containerPolicies:
    # Web container policy
    - containerName: web
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "1000m"
        memory: "2Gi"
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
    
    # Sidecar container policy
    - containerName: sidecar
      mode: "Off"             # Don't autoscale this container
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "200m"
        memory: "256Mi"
```

---

## 🔹 Cluster Autoscaler

### Cluster Autoscaler Overview

**Cluster Autoscaler** automatically adjusts the number of nodes in a cluster based on pod scheduling requirements.

### How Cluster Autoscaler Works

```yaml
# Scale Up Triggers:
- Pods in Pending state due to insufficient resources
- Node utilization below threshold for scale-down-delay

# Scale Down Triggers:
- Node utilization below threshold (default 50%)
- All pods can be moved to other nodes
- No scale-down-disabled annotation
```

### Cluster Autoscaler Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --scale-down-enabled=true
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
        - --scale-down-utilization-threshold=0.5
        - --max-node-provision-time=15m
        
        env:
        - name: AWS_REGION
          value: us-west-2
        
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
```

### Node Pool Annotations

```yaml
# Prevent scale down
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/scale-down-disabled: "true"

# Safe to evict (for pods)
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
```

---

## 🔹 Custom Metrics & External Metrics

### Custom Metrics API

```yaml
# Prometheus Adapter for custom metrics
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'
```

### External Metrics Examples

#### 1. AWS CloudWatch Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sqs-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: sqs_messages_visible
        selector:
          matchLabels:
            queue: "worker-queue"
            region: "us-west-2"
      target:
        type: Value
        value: "100"
```

#### 2. Prometheus Metrics

```yaml
metrics:
- type: External
  external:
    metric:
      name: prometheus_http_requests_rate
      selector:
        matchLabels:
          service: "web-app"
    target:
      type: AverageValue
      averageValue: "100"
```

---

## 🔹 Practical Examples

### 1. Web Application with Multi-Metric HPA

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
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
        ports:
        - containerPort: 80
---
# HPA with multiple metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 100
  metrics:
  # CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metric: requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### 2. Microservice with VPA and HPA

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: microservice
  template:
    metadata:
      labels:
        app: microservice
    spec:
      containers:
      - name: app
        image: microservice:v1.0
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
---
# VPA for right-sizing
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: microservice-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: microservice
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "2000m"
        memory: "4Gi"
---
# HPA for scaling out
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: microservice-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: microservice
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 75
```

### 3. Queue Worker with External Metrics

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue-worker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: queue-worker
  template:
    metadata:
      labels:
        app: queue-worker
    spec:
      containers:
      - name: worker
        image: worker:v1.0
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
---
# HPA based on queue length
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-worker
  minReplicas: 1
  maxReplicas: 50
  metrics:
  # Scale based on SQS queue length
  - type: External
    external:
      metric:
        name: sqs_messages_visible
        selector:
          matchLabels:
            queue: "work-queue"
      target:
        type: AverageValue
        averageValue: "10"  # 10 messages per pod
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods
        value: 5
        periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

---

## 🔹 Best Practices

### 1. Resource Requests and Limits

```yaml
# Always set resource requests for HPA
resources:
  requests:
    cpu: "200m"      # Required for CPU-based HPA
    memory: "256Mi"  # Required for memory-based HPA
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### 2. HPA Configuration

```yaml
# Conservative scaling settings
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60    # Prevent rapid scaling
    policies:
    - type: Percent
      value: 50                       # Scale up by 50% max
      periodSeconds: 60
  
  scaleDown:
    stabilizationWindowSeconds: 300   # 5-minute cooldown
    policies:
    - type: Percent
      value: 10                       # Scale down by 10% max
      periodSeconds: 60
```

### 3. Monitoring and Alerting

```yaml
# Monitor HPA events
kubectl get events --field-selector involvedObject.name=my-hpa

# Set up alerts for:
- HPA scaling events
- Max replicas reached
- Metrics unavailable
- Scaling failures
```

### 4. Testing Autoscaling

```bash
# Load testing
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh
# Inside the pod:
while true; do wget -q -O- http://web-service/; done

# Monitor scaling
kubectl get hpa -w
kubectl top pods
```

---

## 🔹 Troubleshooting

### Common HPA Issues

#### 1. HPA Not Scaling

```bash
# Check HPA status
kubectl describe hpa my-hpa

# Check metrics availability
kubectl top pods
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods

# Common issues:
# - Missing resource requests
# - Metrics server not running
# - Insufficient permissions
```

#### 2. Metrics Not Available

```bash
# Check metrics server
kubectl get pods -n kube-system | grep metrics-server

# Check custom metrics API
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1

# Check external metrics API
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1
```

#### 3. VPA Issues

```bash
# Check VPA status
kubectl describe vpa my-vpa

# Check VPA recommendations
kubectl get vpa my-vpa -o yaml

# Common issues:
# - Insufficient historical data
# - Resource policy conflicts
# - Update mode misconfiguration
```

### Debugging Commands

```bash
# HPA debugging
kubectl get hpa
kubectl describe hpa <name>
kubectl get hpa <name> -o yaml

# Check current metrics
kubectl top pods
kubectl top nodes

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# VPA debugging
kubectl get vpa
kubectl describe vpa <name>

# Cluster Autoscaler logs
kubectl logs -n kube-system deployment/cluster-autoscaler
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: Explain the difference between HPA, VPA, and Cluster Autoscaler.**
- **HPA**: Scales pod replicas horizontally based on metrics
- **VPA**: Scales pod resources vertically (CPU/memory requests)
- **Cluster Autoscaler**: Scales cluster nodes based on pod scheduling needs

**Q2: Can you use HPA and VPA together?**
- Generally not recommended on the same deployment
- VPA restarts pods, which can interfere with HPA
- Use VPA for right-sizing, then HPA for scaling

**Q3: What metrics can HPA use for scaling decisions?**
- Resource metrics (CPU, memory)
- Pod metrics (custom application metrics)
- Object metrics (service/ingress metrics)
- External metrics (cloud provider metrics)

### Technical Questions

**Q4: Why do pods need resource requests for HPA?**
- HPA calculates utilization as: current_usage / requested_resources
- Without requests, utilization calculation is impossible
- Requests provide baseline for percentage calculations

**Q5: How does HPA handle multiple metrics?**
- Calculates desired replicas for each metric
- Uses the highest value (most conservative scaling)
- Ensures all metrics are satisfied

**Q6: What is the HPA algorithm?**
```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

### Scenario-Based Questions

**Q7: Design autoscaling for a web application with traffic spikes.**
```yaml
# Use HPA with:
# - CPU and memory metrics
# - Custom request rate metrics
# - Conservative scale-down behavior
# - Aggressive scale-up for traffic spikes
# - Set appropriate min/max replicas
```

**Q8: How would you handle a queue processing system?**
```yaml
# Use external metrics HPA:
# - Queue length as primary metric
# - Scale based on messages per pod
# - Fast scale-up for queue buildup
# - Slow scale-down to handle bursts
```

**Q9: Troubleshoot HPA not scaling despite high CPU.**
```bash
# Check resource requests are set
# Verify metrics server is running
# Check HPA conditions and events
# Validate target utilization thresholds
# Check for scaling policies restrictions
```

### Best Practices Questions

**Q10: What are autoscaling best practices?**
- Set appropriate resource requests/limits
- Use multiple metrics for better decisions
- Configure conservative scaling behavior
- Monitor and alert on scaling events
- Test autoscaling under load
- Consider application startup time in scaling decisions
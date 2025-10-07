# 🚀 Kubernetes Pods, ReplicaSet & Deployment - DevOps Interview Guide

## Table of Contents
1. [Pods Overview](#pods-overview)
2. [Pod Lifecycle](#pod-lifecycle)
3. [ReplicaSet Overview](#replicaset-overview)
4. [Deployment Overview](#deployment-overview)
5. [Deployment Strategies](#deployment-strategies)
6. [Pod Management](#pod-management)
7. [Practical Examples](#practical-examples)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)
10. [Interview Questions](#interview-questions)

---

## 🔹 Pods Overview

### What is a Pod?

**Pod** is the smallest deployable unit in Kubernetes, representing one or more containers that share storage, network, and lifecycle.

### Pod Characteristics

```yaml
# Pod Key Features:
Shared Resources:
  - Network namespace (IP address, ports)
  - Storage volumes
  - Process namespace (optional)

Atomic Unit:
  - Containers in pod start/stop together
  - Scheduled as single unit
  - Share pod lifecycle

Ephemeral Nature:
  - Pods are mortal (can die and be recreated)
  - Pod IP addresses are not persistent
  - Data is lost unless stored in volumes
```

### Basic Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
  namespace: default
  labels:
    app: web-server
    version: v1.0
    environment: production
  annotations:
    description: "Simple web server pod"
    created-by: "devops-team"
spec:
  # Container specifications
  containers:
  - name: web-container
    image: nginx:1.21
    ports:
    - containerPort: 80
      name: http
      protocol: TCP
    
    # Resource requirements
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    
    # Environment variables
    env:
    - name: ENV_VAR
      value: "production"
    - name: SECRET_KEY
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: secret-key
    
    # Volume mounts
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  
  # Volumes
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  
  # Pod-level settings
  restartPolicy: Always
  terminationGracePeriodSeconds: 30
  dnsPolicy: ClusterFirst
```

### Multi-Container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  # Main application container
  - name: web-app
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
    - name: log-volume
      mountPath: /var/log/nginx
  
  # Sidecar container for log processing
  - name: log-processor
    image: fluentd:v1.14
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nginx
      readOnly: true
    env:
    - name: FLUENTD_CONF
      value: "fluent.conf"
  
  # Init container for setup
  initContainers:
  - name: init-setup
    image: busybox:1.35
    command: ['sh', '-c']
    args:
    - |
      echo "Initializing application..."
      echo "<h1>Hello World</h1>" > /shared/index.html
      echo "Setup complete"
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  
  volumes:
  - name: shared-data
    emptyDir: {}
  - name: log-volume
    emptyDir: {}
```

---

## 🔹 Pod Lifecycle

### Pod Phases

```yaml
# Pod Phases:
Pending: Pod accepted but not yet scheduled/started
Running: Pod bound to node, at least one container running
Succeeded: All containers terminated successfully
Failed: All containers terminated, at least one failed
Unknown: Pod state cannot be determined
```

### Container States

```yaml
# Container States:
Waiting: Container not running (pulling image, etc.)
Running: Container executing without issues
Terminated: Container finished execution or failed
```

### Pod Conditions

```yaml
# Pod Conditions:
PodScheduled: Pod has been scheduled to a node
Initialized: All init containers completed successfully
ContainersReady: All containers in pod are ready
Ready: Pod is ready to serve requests
```

### Pod with Lifecycle Hooks

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    
    # Lifecycle hooks
    lifecycle:
      postStart:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            echo "Container started at $(date)" >> /var/log/lifecycle.log
            # Warm up application
            sleep 5
      
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            echo "Container stopping at $(date)" >> /var/log/lifecycle.log
            # Graceful shutdown
            nginx -s quit
    
    # Health checks
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
    
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    
    # Startup probe for slow-starting containers
    startupProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 30
```

---

## 🔹 ReplicaSet Overview

### What is a ReplicaSet?

**ReplicaSet** ensures a specified number of pod replicas are running at any given time, providing high availability and load distribution.

### ReplicaSet vs Pod

| Aspect | Pod | ReplicaSet |
|--------|-----|------------|
| **Purpose** | Single instance | Multiple replicas |
| **High Availability** | No | Yes |
| **Self-healing** | No | Yes |
| **Scaling** | Manual | Automatic |
| **Management** | Direct | Controller-based |

### Basic ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-replicaset
  labels:
    app: web-server
    tier: frontend
spec:
  # Number of desired replicas
  replicas: 3
  
  # Pod selector
  selector:
    matchLabels:
      app: web-server
      tier: frontend
  
  # Pod template
  template:
    metadata:
      labels:
        app: web-server
        tier: frontend
    spec:
      containers:
      - name: web-container
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### Advanced ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: advanced-replicaset
  labels:
    app: web-app
    version: v2.0
spec:
  replicas: 5
  
  # Advanced selector with expressions
  selector:
    matchLabels:
      app: web-app
    matchExpressions:
    - key: version
      operator: In
      values: ["v2.0", "v2.1"]
    - key: environment
      operator: NotIn
      values: ["development"]
  
  template:
    metadata:
      labels:
        app: web-app
        version: v2.0
        environment: production
    spec:
      containers:
      - name: web
        image: nginx:1.21
        ports:
        - containerPort: 80
        
        # Environment variables
        env:
        - name: REPLICA_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        
        # Volume mounts
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      
      volumes:
      - name: config
        configMap:
          name: nginx-config
```

---

## 🔹 Deployment Overview

### What is a Deployment?

**Deployment** provides declarative updates for pods and ReplicaSets, managing rollouts, rollbacks, and scaling operations.

### Deployment vs ReplicaSet

| Feature | ReplicaSet | Deployment |
|---------|------------|------------|
| **Updates** | Manual | Declarative |
| **Rollback** | Manual | Automatic |
| **Rolling Updates** | No | Yes |
| **History** | No | Yes |
| **Pause/Resume** | No | Yes |

### Basic Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web-server
spec:
  # Replica configuration
  replicas: 3
  
  # Update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  
  # Pod selector
  selector:
    matchLabels:
      app: web-server
  
  # Pod template
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: web
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### Complete Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-deployment
  namespace: production
  labels:
    app: web-application
    version: v1.0
    environment: production
  annotations:
    deployment.kubernetes.io/revision: "1"
    description: "Production web application deployment"
spec:
  # Replica management
  replicas: 5
  
  # Revision history
  revisionHistoryLimit: 10
  
  # Progress deadline
  progressDeadlineSeconds: 600
  
  # Minimum ready seconds
  minReadySeconds: 30
  
  # Update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  
  # Pod selector
  selector:
    matchLabels:
      app: web-application
      environment: production
  
  # Pod template
  template:
    metadata:
      labels:
        app: web-application
        version: v1.0
        environment: production
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # Service account
      serviceAccountName: web-app-sa
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      # Init containers
      initContainers:
      - name: init-db
        image: busybox:1.35
        command: ['sh', '-c']
        args:
        - |
          until nslookup database-service; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready!"
      
      # Main containers
      containers:
      - name: web-app
        image: myapp:v1.0
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
        
        # Environment variables
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log-level
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        # Resource requirements
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
            ephemeral-storage: "1Gi"
          limits:
            cpu: "500m"
            memory: "512Mi"
            ephemeral-storage: "2Gi"
        
        # Health checks
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 30
        
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
          failureThreshold: 3
        
        # Volume mounts
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        - name: data-volume
          mountPath: /app/data
        - name: tmp-volume
          mountPath: /tmp
      
      # Sidecar container
      - name: log-shipper
        image: fluentd:v1.14
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
        volumeMounts:
        - name: log-volume
          mountPath: /var/log
      
      # Volumes
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: data-volume
        persistentVolumeClaim:
          claimName: app-data-pvc
      - name: tmp-volume
        emptyDir: {}
      - name: log-volume
        emptyDir: {}
      
      # Pod settings
      restartPolicy: Always
      terminationGracePeriodSeconds: 60
      dnsPolicy: ClusterFirst
      
      # Node selection
      nodeSelector:
        disktype: ssd
      
      # Tolerations
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "web-app"
        effect: "NoSchedule"
      
      # Affinity rules
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: web-application
              topologyKey: kubernetes.io/hostname
```

---

## 🔹 Deployment Strategies

### Rolling Update (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%    # Max pods unavailable during update
      maxSurge: 25%          # Max pods above desired replica count
  minReadySeconds: 30        # Wait time before considering pod ready
```

### Recreate Strategy

```yaml
spec:
  strategy:
    type: Recreate    # Kill all pods, then create new ones
  # Use for:
  # - Applications that can't run multiple versions
  # - Shared storage that doesn't support concurrent access
  # - Stateful applications requiring clean shutdown
```

### Blue-Green Deployment Pattern

```yaml
# Blue Deployment (Current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1.0
---
# Green Deployment (New)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v2.0
---
# Service (switch selector to change traffic)
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
    version: blue    # Change to 'green' for deployment
  ports:
  - port: 80
    targetPort: 8080
```

### Canary Deployment Pattern

```yaml
# Main Deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-main
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: app
        image: myapp:v1.0
---
# Canary Deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: app
        image: myapp:v2.0
---
# Service (routes to both deployments)
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp    # Matches both stable and canary
  ports:
  - port: 80
    targetPort: 8080
```

---

## 🔹 Pod Management

### Pod Operations

```bash
# Create pod
kubectl apply -f pod.yaml

# List pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels

# Describe pod
kubectl describe pod my-pod

# Get pod logs
kubectl logs my-pod
kubectl logs my-pod -c container-name
kubectl logs my-pod --previous

# Execute commands in pod
kubectl exec -it my-pod -- /bin/bash
kubectl exec my-pod -- ls -la /app

# Port forwarding
kubectl port-forward pod/my-pod 8080:80

# Delete pod
kubectl delete pod my-pod
```

### ReplicaSet Operations

```bash
# Create ReplicaSet
kubectl apply -f replicaset.yaml

# List ReplicaSets
kubectl get replicasets
kubectl get rs

# Scale ReplicaSet
kubectl scale replicaset my-rs --replicas=5

# Delete ReplicaSet
kubectl delete replicaset my-rs
kubectl delete rs my-rs --cascade=orphan  # Keep pods
```

### Deployment Operations

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.21
kubectl apply -f deployment.yaml

# List deployments
kubectl get deployments
kubectl get deploy

# Scale deployment
kubectl scale deployment my-deploy --replicas=10

# Update deployment
kubectl set image deployment/my-deploy container=nginx:1.22
kubectl patch deployment my-deploy -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}}}'

# Rollout management
kubectl rollout status deployment/my-deploy
kubectl rollout history deployment/my-deploy
kubectl rollout undo deployment/my-deploy
kubectl rollout undo deployment/my-deploy --to-revision=2

# Pause/Resume rollout
kubectl rollout pause deployment/my-deploy
kubectl rollout resume deployment/my-deploy

# Delete deployment
kubectl delete deployment my-deploy
```

---

## 🔹 Practical Examples

### 1. Web Application with Database

```yaml
# Database Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: "webapp"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-pvc
---
# Web Application Deployment
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
      initContainers:
      - name: wait-for-db
        image: busybox:1.35
        command: ['sh', '-c']
        args:
        - |
          until nc -z postgres-service 5432; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready!"
      containers:
      - name: web
        image: webapp:v1.0
        env:
        - name: DATABASE_URL
          value: "postgresql://$(DB_USER):$(DB_PASS)@postgres-service:5432/webapp"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### 2. Microservices Architecture

```yaml
# User Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: user-service:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "user-service"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: user-db-secret
              key: url
---
# Order Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "order-service"
        - name: USER_SERVICE_URL
          value: "http://user-service:8080"
---
# API Gateway
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: gateway
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: nginx-config
        configMap:
          name: gateway-config
```

### 3. Batch Processing Job

```yaml
# Job Processor Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: job-processor
spec:
  replicas: 5
  selector:
    matchLabels:
      app: job-processor
  template:
    metadata:
      labels:
        app: job-processor
    spec:
      containers:
      - name: processor
        image: job-processor:v1.0
        env:
        - name: QUEUE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: queue-url
        - name: WORKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "pgrep -f processor"
          initialDelaySeconds: 30
          periodSeconds: 30
```

---

## 🔹 Best Practices

### 1. Resource Management

```yaml
# Always set resource requests and limits
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# Use appropriate QoS classes
# Guaranteed: requests = limits
# Burstable: requests < limits  
# BestEffort: no requests/limits
```

### 2. Health Checks

```yaml
# Implement comprehensive health checks
startupProbe:    # For slow-starting containers
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30

livenessProbe:   # Restart unhealthy containers
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 30

readinessProbe:  # Remove from service when not ready
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 10
```

### 3. Security Considerations

```yaml
# Security best practices
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL

# Use service accounts
serviceAccountName: app-service-account

# Implement network policies
# Use secrets for sensitive data
# Regular security scanning
```

### 4. Deployment Strategies

```yaml
# Use rolling updates for zero downtime
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%

# Set appropriate readiness checks
minReadySeconds: 30

# Keep deployment history
revisionHistoryLimit: 10
```

---

## 🔹 Troubleshooting

### Common Pod Issues

#### 1. Pod Stuck in Pending

```bash
# Check pod events
kubectl describe pod my-pod

# Common causes:
# - Insufficient cluster resources
# - Node selector constraints
# - Persistent volume issues
# - Image pull secrets missing
```

#### 2. Pod CrashLoopBackOff

```bash
# Check pod logs
kubectl logs my-pod --previous

# Check resource limits
kubectl describe pod my-pod | grep -A 5 Limits

# Common causes:
# - Application startup failures
# - Resource constraints
# - Missing dependencies
# - Configuration errors
```

#### 3. Pod Not Ready

```bash
# Check readiness probe
kubectl describe pod my-pod | grep -A 10 Readiness

# Check service endpoints
kubectl get endpoints my-service

# Common causes:
# - Readiness probe failures
# - Application not fully started
# - Dependencies unavailable
```

### Debugging Commands

```bash
# Pod debugging
kubectl get pods -o wide
kubectl describe pod my-pod
kubectl logs my-pod -f
kubectl exec -it my-pod -- /bin/bash

# Deployment debugging
kubectl get deployments
kubectl describe deployment my-deploy
kubectl rollout status deployment/my-deploy
kubectl get replicasets

# Events and troubleshooting
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: What's the difference between Pod, ReplicaSet, and Deployment?**
- **Pod**: Single instance, no self-healing
- **ReplicaSet**: Multiple replicas, self-healing, no updates
- **Deployment**: Declarative updates, rollbacks, rolling updates

**Q2: Why do we need ReplicaSets when we have Deployments?**
- Deployments manage ReplicaSets
- ReplicaSets provide the actual replica management
- Deployments add update and rollback capabilities

**Q3: What happens when a pod dies?**
- If managed by ReplicaSet/Deployment: automatically recreated
- If standalone pod: remains dead until manually recreated
- Data in containers is lost unless stored in volumes

### Technical Questions

**Q4: How do you perform a rolling update?**
```bash
kubectl set image deployment/my-app container=new-image:tag
kubectl rollout status deployment/my-app
```

**Q5: How do you rollback a deployment?**
```bash
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=2
```

**Q6: What are the different restart policies?**
- **Always**: Always restart (default for Deployments)
- **OnFailure**: Restart only on failure (good for Jobs)
- **Never**: Never restart

### Scenario-Based Questions

**Q7: Design a highly available web application.**
```yaml
# Use Deployment with multiple replicas
# Implement pod anti-affinity
# Set resource requests and limits
# Configure health checks
# Use rolling update strategy
```

**Q8: How would you troubleshoot a failing deployment?**
```bash
# Check deployment status
# Review pod events and logs
# Verify resource availability
# Check image and configuration
# Review health check configuration
```

**Q9: Implement zero-downtime deployment.**
```yaml
# Use rolling update strategy
# Configure readiness probes
# Set minReadySeconds
# Ensure sufficient cluster resources
# Test deployment process
```

### Best Practices Questions

**Q10: What are pod and deployment best practices?**
- Always set resource requests and limits
- Implement comprehensive health checks
- Use appropriate security contexts
- Follow the principle of least privilege
- Use meaningful labels and annotations
- Implement proper logging and monitoring
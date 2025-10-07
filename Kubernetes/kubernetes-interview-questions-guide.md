# 🚀 Kubernetes Interview Questions & Solutions - Senior DevOps Guide

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Workloads & Controllers](#workloads--controllers)
3. [Services & Networking](#services--networking)
4. [Storage & Volumes](#storage--volumes)
5. [Configuration Management](#configuration-management)
6. [Security](#security)
7. [Monitoring & Logging](#monitoring--logging)
8. [Troubleshooting](#troubleshooting)
9. [Advanced Topics](#advanced-topics)
10. [Best Practices](#best-practices)

---

## 🔹 Core Concepts

### Q1: Explain Kubernetes architecture and its main components.

**Answer**: Kubernetes follows a master-worker architecture with control plane and worker nodes.

**Control Plane Components**:
```yaml
# Control Plane Architecture
API Server (kube-apiserver):
  - REST API endpoint for all cluster operations
  - Authentication, authorization, admission control
  - Validates and processes API requests

etcd:
  - Distributed key-value store
  - Stores all cluster data and state
  - Provides consistency and high availability

Controller Manager (kube-controller-manager):
  - Runs controller processes
  - Node controller, Replication controller, etc.
  - Watches for changes and maintains desired state

Scheduler (kube-scheduler):
  - Assigns pods to nodes
  - Considers resource requirements, constraints
  - Makes optimal placement decisions
```

**Worker Node Components**:
```yaml
# Worker Node Architecture
kubelet:
  - Primary node agent
  - Manages pod lifecycle
  - Communicates with API server
  - Ensures containers are running

kube-proxy:
  - Network proxy on each node
  - Maintains network rules
  - Enables service communication
  - Load balances traffic

Container Runtime:
  - Runs containers (Docker, containerd, CRI-O)
  - Pulls images and manages container lifecycle
```

**Example - Checking Components**:
```bash
# Check control plane components
kubectl get componentstatuses
kubectl get pods -n kube-system

# Check node status
kubectl get nodes -o wide
kubectl describe node <node-name>
```

---

### Q2: What is the difference between a Pod and a Container?

**Answer**: Pod is the smallest deployable unit in Kubernetes, while containers run inside pods.

**Key Differences**:
```yaml
Container:
  - Single application process
  - Isolated runtime environment
  - Has its own filesystem and process space

Pod:
  - Wrapper around one or more containers
  - Shared network namespace (IP address)
  - Shared storage volumes
  - Atomic unit of deployment
```

**Example - Multi-container Pod**:
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
  
  # Sidecar container
  - name: log-processor
    image: fluentd:v1.14
    volumeMounts:
    - name: shared-data
      mountPath: /var/log
  
  # Init container
  initContainers:
  - name: init-setup
    image: busybox:1.35
    command: ['sh', '-c', 'echo "Hello World" > /shared/index.html']
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  
  volumes:
  - name: shared-data
    emptyDir: {}
```

**When to use multi-container pods**:
- Sidecar pattern (logging, monitoring)
- Ambassador pattern (proxy)
- Adapter pattern (data transformation)

---

### Q3: Explain Kubernetes namespaces and their use cases.

**Answer**: Namespaces provide virtual clusters within a physical cluster for resource isolation.

**Use Cases**:
```yaml
# Different environments
Development: dev-namespace
Staging: staging-namespace
Production: prod-namespace

# Team isolation
Team-A: team-a-namespace
Team-B: team-b-namespace

# Application separation
Frontend: frontend-namespace
Backend: backend-namespace
Database: database-namespace
```

**Example - Namespace Management**:
```bash
# Create namespace
kubectl create namespace production

# Set default namespace
kubectl config set-context --current --namespace=production

# List resources in specific namespace
kubectl get pods -n production
kubectl get all -n production

# Cross-namespace communication
# service-name.namespace.svc.cluster.local
curl http://api-service.backend.svc.cluster.local:8080
```

**Namespace with Resource Quotas**:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

---

## 🔹 Workloads & Controllers

### Q4: Compare Deployment, StatefulSet, and DaemonSet. When would you use each?

**Answer**: Different controllers for different application patterns and requirements.

**Deployment**:
```yaml
# Use for: Stateless applications, web servers, APIs
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
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

**StatefulSet**:
```yaml
# Use for: Databases, stateful applications requiring persistent identity
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
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
          value: production
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

**DaemonSet**:
```yaml
# Use for: Node-level services, monitoring agents, log collectors
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      tolerations:
      - operator: Exists
        effect: NoSchedule
```

**Key Differences**:
| Feature | Deployment | StatefulSet | DaemonSet |
|---------|------------|-------------|-----------|
| **Pod Identity** | Random | Stable, ordered | One per node |
| **Storage** | Shared | Persistent per pod | Usually host-mounted |
| **Scaling** | Parallel | Ordered | Node-based |
| **Use Case** | Stateless apps | Databases | System services |

---

### Q5: How do you implement rolling updates and rollbacks?

**Answer**: Use Deployment strategies for controlled updates and easy rollbacks.

**Rolling Update Strategy**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%  # Max pods unavailable during update
      maxSurge: 25%        # Max extra pods during update
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
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Update Commands**:
```bash
# Update image
kubectl set image deployment/web-app web=nginx:1.22

# Check rollout status
kubectl rollout status deployment/web-app

# View rollout history
kubectl rollout history deployment/web-app

# Rollback to previous version
kubectl rollout undo deployment/web-app

# Rollback to specific revision
kubectl rollout undo deployment/web-app --to-revision=2

# Pause/resume rollout
kubectl rollout pause deployment/web-app
kubectl rollout resume deployment/web-app
```

**Blue-Green Deployment**:
```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-blue
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web-app
      version: blue
  template:
    metadata:
      labels:
        app: web-app
        version: blue
    spec:
      containers:
      - name: web
        image: nginx:1.21
---
# Green deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-green
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web-app
      version: green
  template:
    metadata:
      labels:
        app: web-app
        version: green
    spec:
      containers:
      - name: web
        image: nginx:1.22
---
# Service switches between blue and green
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
    version: blue  # Switch to 'green' for deployment
  ports:
  - port: 80
    targetPort: 80
```

---

## 🔹 Services & Networking

### Q6: Explain different Service types and their use cases.

**Answer**: Services provide stable network endpoints for accessing pods.

**ClusterIP (Default)**:
```yaml
# Internal cluster communication only
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```

**NodePort**:
```yaml
# External access via node IP:port
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 30000-32767 range
```

**LoadBalancer**:
```yaml
# Cloud provider load balancer
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
```

**ExternalName**:
```yaml
# DNS alias for external services
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
  ports:
  - port: 5432
```

**Headless Service**:
```yaml
# Direct pod access, no load balancing
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

---

### Q7: How does Kubernetes networking work? Explain pod-to-pod communication.

**Answer**: Kubernetes uses a flat network model where every pod gets a unique IP.

**Network Model Requirements**:
```yaml
# Kubernetes Network Model
Pod-to-Pod Communication:
  - Every pod gets unique IP address
  - Pods can communicate without NAT
  - Cross-node communication works seamlessly

Service Discovery:
  - Services provide stable endpoints
  - DNS-based service discovery
  - kube-dns/CoreDNS for name resolution

Network Policies:
  - Control traffic flow between pods
  - Ingress and egress rules
  - Label-based selection
```

**CNI (Container Network Interface)**:
```bash
# Popular CNI plugins
Flannel: Simple overlay network
Calico: Network policies + routing
Weave: Mesh networking
Cilium: eBPF-based networking

# Check CNI configuration
ls /etc/cni/net.d/
cat /etc/cni/net.d/10-flannel.conflist
```

**Pod Communication Example**:
```yaml
# Frontend pod communicating with backend
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: web
    image: nginx:1.21
    env:
    - name: BACKEND_URL
      value: "http://backend-service:8080"  # Service DNS name
---
# Backend service
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```

**DNS Resolution**:
```bash
# Service DNS format
<service-name>.<namespace>.svc.cluster.local

# Examples
backend-service.production.svc.cluster.local
database.default.svc.cluster.local

# Test DNS resolution
kubectl exec -it frontend-pod -- nslookup backend-service
kubectl exec -it frontend-pod -- dig backend-service.production.svc.cluster.local
```

---

## 🔹 Storage & Volumes

### Q8: Explain Persistent Volumes, Persistent Volume Claims, and Storage Classes.

**Answer**: Kubernetes storage abstraction for managing persistent data.

**Storage Architecture**:
```yaml
# Storage Components
Persistent Volume (PV):
  - Cluster-level storage resource
  - Independent of pod lifecycle
  - Provisioned by admin or dynamically

Persistent Volume Claim (PVC):
  - Request for storage by user
  - Binds to available PV
  - Used by pods to mount storage

Storage Class:
  - Defines storage types
  - Enables dynamic provisioning
  - Specifies provisioner and parameters
```

**Storage Class**:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

**Persistent Volume**:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  awsElasticBlockStore:
    volumeID: vol-12345678
    fsType: ext4
```

**Persistent Volume Claim**:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
```

**Using PVC in Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:13
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
```

**Volume Types**:
```yaml
# Different volume types
emptyDir: {}              # Temporary storage
hostPath:                 # Node filesystem
  path: /data
configMap:                # Configuration data
  name: app-config
secret:                   # Sensitive data
  secretName: app-secret
nfs:                      # Network File System
  server: nfs-server.example.com
  path: /path/to/data
```

---

## 🔹 Configuration Management

### Q9: How do you manage configuration and secrets in Kubernetes?

**Answer**: Use ConfigMaps for configuration and Secrets for sensitive data.

**ConfigMap**:
```yaml
# ConfigMap for application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Key-value pairs
  database.host: "postgres-service"
  database.port: "5432"
  log.level: "INFO"
  
  # File content
  application.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://postgres-service:5432/mydb
    logging.level.com.example=DEBUG
  
  nginx.conf: |
    server {
        listen 80;
        location / {
            proxy_pass http://backend:8080;
        }
    }
```

**Using ConfigMap in Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    
    # Environment variables from ConfigMap
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.host
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.port
    
    # All ConfigMap data as env vars
    envFrom:
    - configMapRef:
        name: app-config
    
    # Mount as files
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
  
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: nginx-config
    configMap:
      name: app-config
```

**Secrets**:
```yaml
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # Base64 encoded values
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
stringData:
  # Plain text (automatically encoded)
  api-key: "abc123def456"
  database-url: "postgresql://user:pass@host:5432/db"
---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...
  tls.key: LS0tLS1CRUdJTi...
---
# Docker registry secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6e...
```

**Using Secrets**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  # Use registry secret for image pull
  imagePullSecrets:
  - name: registry-secret
  
  containers:
  - name: app
    image: private-registry.com/myapp:latest
    
    # Environment variables from secret
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: password
    
    # Mount secret as files
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
      defaultMode: 0400
```

---

## 🔹 Security

### Q10: How do you implement security in Kubernetes?

**Answer**: Multi-layered security approach with RBAC, security contexts, and network policies.

**RBAC (Role-Based Access Control)**:
```yaml
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
---
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Security Contexts**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod-level security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: app
    image: nginx:1.21
    
    # Container-level security context
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
    
    # Writable volumes for read-only filesystem
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
```

**Network Policies**:
```yaml
# Default deny all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow specific traffic
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

**Pod Security Standards**:
```yaml
# Namespace with Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## 🔹 Monitoring & Logging

### Q11: How do you implement monitoring and logging in Kubernetes?

**Answer**: Use Prometheus for metrics, Grafana for visualization, and centralized logging.

**Prometheus Monitoring**:
```yaml
# ServiceMonitor for Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
spec:
  selector:
    matchLabels:
      app: web-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
---
# Application with metrics endpoint
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    metadata:
      labels:
        app: web-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
```

**Alerting Rules**:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
spec:
  groups:
  - name: app.rules
    rules:
    - alert: HighCPUUsage
      expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
        description: "CPU usage is above 80% for {{ $labels.pod }}"
    
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod is crash looping"
        description: "Pod {{ $labels.pod }} is restarting frequently"
```

**Centralized Logging**:
```yaml
# Fluentd DaemonSet for log collection
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.15-debian-elasticsearch7-1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

---

## 🔹 Troubleshooting

### Q12: How do you troubleshoot common Kubernetes issues?

**Answer**: Systematic approach using kubectl commands and log analysis.

**Pod Issues**:
```bash
# Check pod status
kubectl get pods -o wide
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous container logs
kubectl logs <pod-name> -c <container-name>  # Multi-container pod

# Debug running pod
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash

# Port forwarding for testing
kubectl port-forward pod/<pod-name> 8080:80
```

**Service Issues**:
```bash
# Check service and endpoints
kubectl get svc
kubectl describe svc <service-name>
kubectl get endpoints <service-name>

# Test service connectivity
kubectl run debug --image=busybox -it --rm -- /bin/sh
# Inside debug pod:
nslookup <service-name>
wget -qO- http://<service-name>:<port>
```

**Node Issues**:
```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>
kubectl top nodes

# Check node conditions
kubectl get nodes -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}'

# Drain node for maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node-name>
```

**Resource Issues**:
```bash
# Check resource usage
kubectl top pods --all-namespaces --sort-by=memory
kubectl top nodes

# Check resource quotas
kubectl describe quota -n <namespace>
kubectl describe limitrange -n <namespace>

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events --field-selector type=Warning
```

**Common Issues and Solutions**:
```yaml
# ImagePullBackOff
Issues:
  - Wrong image name/tag
  - Missing image pull secrets
  - Registry authentication

Solutions:
  - Verify image exists
  - Check imagePullSecrets
  - Test registry access

# CrashLoopBackOff
Issues:
  - Application startup failure
  - Resource constraints
  - Configuration errors

Solutions:
  - Check application logs
  - Verify resource limits
  - Review configuration

# Pending Pods
Issues:
  - Insufficient resources
  - Node selector constraints
  - PVC binding issues

Solutions:
  - Check node resources
  - Review scheduling constraints
  - Verify storage availability
```

This comprehensive guide covers all essential Kubernetes concepts for senior DevOps engineer interviews, with practical examples and real-world scenarios.
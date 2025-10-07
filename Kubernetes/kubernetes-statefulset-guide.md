# 🗄️ Kubernetes StatefulSet - DevOps Interview Guide

## Table of Contents
1. [StatefulSet Overview](#statefulset-overview)
2. [Core Concepts](#core-concepts)
3. [StatefulSet Manifest](#statefulset-manifest)
4. [Persistent Storage](#persistent-storage)
5. [Scaling and Updates](#scaling-and-updates)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Interview Questions](#interview-questions)

---

## 🔹 StatefulSet Overview

### What is a StatefulSet?

**StatefulSet** manages stateful applications that require stable network identities, persistent storage, and ordered deployment/scaling.

### StatefulSet vs Deployment

| Aspect | StatefulSet | Deployment |
|--------|-------------|------------|
| **Pod Identity** | Stable, unique (web-0, web-1) | Random names |
| **Network Identity** | Stable hostname/DNS | Dynamic |
| **Storage** | Persistent per pod | Shared or ephemeral |
| **Scaling** | Ordered (sequential) | Parallel |
| **Updates** | Ordered rolling updates | Parallel updates |
| **Use Cases** | Databases, message queues | Stateless web apps |

### Key Features

```yaml
# StatefulSet Guarantees:
1. Stable Network Identity - Predictable pod names and DNS
2. Stable Storage - Persistent volumes per pod
3. Ordered Deployment - Pods created sequentially (0, 1, 2...)
4. Ordered Scaling - Scale up/down one pod at a time
5. Ordered Updates - Rolling updates in reverse order
```

---

## 🔹 Core Concepts

### Pod Naming and Identity

```yaml
# StatefulSet: web
# Pods created:
web-0  # First pod (ordinal 0)
web-1  # Second pod (ordinal 1)
web-2  # Third pod (ordinal 2)

# DNS Names (with headless service):
web-0.web-service.default.svc.cluster.local
web-1.web-service.default.svc.cluster.local
web-2.web-service.default.svc.cluster.local
```

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  clusterIP: None        # Headless service
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

### Deployment and Scaling Order

```yaml
# Scale Up (0 → 3):
1. Create web-0, wait for Running and Ready
2. Create web-1, wait for Running and Ready  
3. Create web-2, wait for Running and Ready

# Scale Down (3 → 1):
1. Delete web-2, wait for termination
2. Delete web-1, wait for termination
3. Keep web-0 running

# Updates (Rolling Update):
1. Update web-2, wait for Ready
2. Update web-1, wait for Ready
3. Update web-0, wait for Ready
```

---

## 🔹 StatefulSet Manifest

### Basic StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: default
  labels:
    app: web
    component: frontend
spec:
  # Service name for DNS
  serviceName: web-service
  
  # Number of replicas
  replicas: 3
  
  # Pod selector
  selector:
    matchLabels:
      app: web
  
  # Pod template
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  
  # Volume claim templates
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
      storageClassName: "fast-ssd"
```

### Complete StatefulSet Configuration

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: production
  labels:
    app: database
    component: postgres
    version: v13
  annotations:
    description: "PostgreSQL StatefulSet with persistent storage"
spec:
  # Service name for stable network identity
  serviceName: postgres-headless
  
  # Replica configuration
  replicas: 3
  
  # Pod management policy
  podManagementPolicy: OrderedReady  # OrderedReady (default) | Parallel
  
  # Update strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0           # Update pods with ordinal >= partition
      maxUnavailable: 1      # Kubernetes 1.24+
  
  # Minimum ready seconds
  minReadySeconds: 10
  
  # Revision history limit
  revisionHistoryLimit: 10
  
  # Pod selector
  selector:
    matchLabels:
      app: database
      component: postgres
  
  # Pod template
  template:
    metadata:
      labels:
        app: database
        component: postgres
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9187"
    spec:
      # Service account
      serviceAccountName: postgres-sa
      
      # Security context
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      
      # Init containers
      initContainers:
      - name: init-postgres
        image: postgres:13
        command:
        - /bin/bash
        - -c
        - |
          if [ ! -f /var/lib/postgresql/data/PG_VERSION ]; then
            echo "Initializing PostgreSQL data directory..."
            initdb -D /var/lib/postgresql/data
          fi
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data
      
      # Main containers
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
          name: postgres
        
        # Environment variables
        env:
        - name: POSTGRES_DB
          value: "myapp"
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
        - name: PGDATA
          value: /var/lib/postgresql/data
        
        # Resource limits
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        
        # Health checks
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Volume mounts
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
          readOnly: true
        - name: postgres-init
          mountPath: /docker-entrypoint-initdb.d
          readOnly: true
      
      # Sidecar container for monitoring
      - name: postgres-exporter
        image: prometheuscommunity/postgres-exporter:v0.10.1
        ports:
        - containerPort: 9187
          name: metrics
        env:
        - name: DATA_SOURCE_NAME
          value: "postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@localhost:5432/$(POSTGRES_DB)?sslmode=disable"
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
        - name: POSTGRES_DB
          value: "myapp"
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
      
      # Volumes
      volumes:
      - name: postgres-config
        configMap:
          name: postgres-config
      - name: postgres-init
        configMap:
          name: postgres-init-scripts
      
      # Termination grace period
      terminationGracePeriodSeconds: 60
      
      # Node selection
      nodeSelector:
        storage-type: ssd
      
      # Tolerations
      tolerations:
      - key: "database"
        operator: "Equal"
        value: "postgres"
        effect: "NoSchedule"
  
  # Persistent volume claim templates
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
      labels:
        app: database
        component: postgres
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
      storageClassName: "fast-ssd"
```

---

## 🔹 Persistent Storage

### Volume Claim Templates

```yaml
# Volume Claim Templates create PVCs for each pod
volumeClaimTemplates:
- metadata:
    name: data
    labels:
      app: database
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 50Gi
    storageClassName: "ssd"

# Results in PVCs:
# data-database-0 (for pod database-0)
# data-database-1 (for pod database-1)
# data-database-2 (for pod database-2)
```

### Storage Classes

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
  encrypted: "true"
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

### Persistent Volume Management

```bash
# List PVCs created by StatefulSet
kubectl get pvc -l app=database

# Check PV status
kubectl get pv

# Describe PVC for specific pod
kubectl describe pvc data-database-0

# Resize PVC (if storage class supports it)
kubectl patch pvc data-database-0 -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

---

## 🔹 Scaling and Updates

### Scaling StatefulSets

```bash
# Scale up (adds pods sequentially)
kubectl scale statefulset database --replicas=5

# Scale down (removes pods in reverse order)
kubectl scale statefulset database --replicas=2

# Check scaling progress
kubectl get pods -l app=database -w
```

### Update Strategies

#### Rolling Update (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0    # Update pods with ordinal >= partition
```

#### Partitioned Updates

```yaml
# Update only pods with ordinal >= 2
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2

# Gradually roll out updates
# 1. Set partition: 2, update image
# 2. Set partition: 1, continue rollout
# 3. Set partition: 0, complete rollout
```

#### OnDelete Update

```yaml
spec:
  updateStrategy:
    type: OnDelete

# Manual update process:
# 1. Update StatefulSet spec
# 2. Delete pods manually to trigger recreation
kubectl delete pod database-2
```

### Update Operations

```bash
# Update container image
kubectl patch statefulset database -p '{"spec":{"template":{"spec":{"containers":[{"name":"postgres","image":"postgres:14"}]}}}}'

# Check rollout status
kubectl rollout status statefulset/database

# View rollout history
kubectl rollout history statefulset/database

# Rollback to previous version
kubectl rollout undo statefulset/database
```

---

## 🔹 Practical Examples

### 1. PostgreSQL Cluster

```yaml
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# Regular Service for external access
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
spec:
  selector:
    app: postgres
    role: primary
  ports:
  - port: 5432
    targetPort: 5432
---
# PostgreSQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
      - name: init-postgres
        image: postgres:13
        command:
        - /bin/bash
        - -c
        - |
          set -ex
          # Generate server id from pod ordinal
          [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          
          # Copy appropriate conf.d files from config-map to emptyDir
          if [[ $ordinal -eq 0 ]]; then
            # Primary configuration
            cp /mnt/config-map/primary.conf /mnt/conf.d/
            echo "role=primary" >> /mnt/conf.d/server.conf
          else
            # Replica configuration
            cp /mnt/config-map/replica.conf /mnt/conf.d/
            echo "role=replica" >> /mnt/conf.d/server.conf
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: "myapp"
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
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        
        ports:
        - containerPort: 5432
          name: postgres
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: conf
          mountPath: /etc/postgresql/conf.d
          readOnly: true
        
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"
          initialDelaySeconds: 30
          periodSeconds: 30
        
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"
          initialDelaySeconds: 5
          periodSeconds: 10
      
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: postgres-config
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
      storageClassName: "fast-ssd"
```

### 2. MongoDB Replica Set

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb-headless
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      initContainers:
      - name: init-mongo
        image: mongo:5.0
        command:
        - /bin/bash
        - -c
        - |
          set -ex
          # Generate machine-id
          cat /proc/sys/kernel/random/uuid > /tmp/machine-id
          
          # Wait for MongoDB to be ready
          until mongo --host mongodb-0.mongodb-headless:27017 --eval "print('ready')"; do
            echo "Waiting for MongoDB..."
            sleep 2
          done
        volumeMounts:
        - name: workdir
          mountPath: /work-dir
      
      containers:
      - name: mongodb
        image: mongo:5.0
        command:
        - mongod
        - --replSet=rs0
        - --bind_ip_all
        - --smallfiles
        - --noprealloc
        
        ports:
        - containerPort: 27017
          name: mongodb
        
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
        
        volumeMounts:
        - name: data
          mountPath: /data/db
        - name: workdir
          mountPath: /work-dir
        
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
        
        livenessProbe:
          exec:
            command:
            - mongo
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 30
        
        readinessProbe:
          exec:
            command:
            - mongo
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 5
          periodSeconds: 10
      
      volumes:
      - name: workdir
        emptyDir: {}
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 50Gi
      storageClassName: "ssd"
```

### 3. Elasticsearch Cluster

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: elasticsearch-headless
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      - name: increase-vm-max-map
        image: busybox:1.35
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      
      containers:
      - name: elasticsearch
        image: elasticsearch:7.17.0
        env:
        - name: cluster.name
          value: "elasticsearch-cluster"
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch-headless,elasticsearch-1.elasticsearch-headless,elasticsearch-2.elasticsearch-headless"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        - name: xpack.security.enabled
          value: "false"
        
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        
        livenessProbe:
          httpGet:
            path: /_cluster/health
            port: 9200
          initialDelaySeconds: 30
          periodSeconds: 30
        
        readinessProbe:
          httpGet:
            path: /_cluster/health?wait_for_status=yellow
            port: 9200
          initialDelaySeconds: 5
          periodSeconds: 10
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
      storageClassName: "fast-ssd"
```

---

## 🔹 Best Practices

### 1. Design Considerations

```yaml
# Stateful Application Requirements:
- Stable network identity needed
- Persistent storage per instance
- Ordered deployment/scaling required
- Data consistency important
- Leader election or clustering

# Use StatefulSet for:
- Databases (PostgreSQL, MySQL, MongoDB)
- Message queues (Kafka, RabbitMQ)
- Distributed systems (Elasticsearch, Cassandra)
- Caching systems (Redis Cluster)
```

### 2. Storage Management

```yaml
# Storage Best Practices:
- Use appropriate storage classes
- Set proper resource requests
- Enable volume expansion if needed
- Implement backup strategies
- Monitor storage usage

# Storage Class Selection:
- SSD for databases requiring IOPS
- Network storage for shared access
- Local storage for performance-critical apps
```

### 3. Scaling Strategies

```yaml
# Scaling Considerations:
- Plan for data rebalancing
- Consider application-specific scaling logic
- Monitor resource usage during scaling
- Test scaling procedures regularly
- Implement proper health checks

# Gradual Scaling:
- Scale one pod at a time
- Wait for readiness before next pod
- Monitor cluster health during scaling
```

### 4. Update Management

```yaml
# Update Strategies:
- Use partitioned updates for gradual rollouts
- Test updates in staging environment
- Implement proper backup before updates
- Monitor application health during updates
- Have rollback procedures ready
```

---

## 🔹 Troubleshooting

### Common Issues

#### 1. Pod Stuck in Pending

```bash
# Check PVC status
kubectl get pvc -l app=myapp

# Check storage class
kubectl describe storageclass fast-ssd

# Check node resources
kubectl describe nodes

# Common causes:
# - No available storage
# - Storage class issues
# - Node selector constraints
# - Resource constraints
```

#### 2. Pods Not Starting in Order

```bash
# Check pod status
kubectl get pods -l app=myapp

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check readiness probes
kubectl describe pod myapp-0

# Common causes:
# - Readiness probe failures
# - Init container issues
# - Resource constraints
```

#### 3. Storage Issues

```bash
# Check PV and PVC status
kubectl get pv,pvc

# Check storage provisioner logs
kubectl logs -n kube-system -l app=ebs-csi-controller

# Check mount issues
kubectl describe pod myapp-0 | grep -A 10 "Volumes:"
```

### Debugging Commands

```bash
# StatefulSet status
kubectl get statefulset
kubectl describe statefulset myapp

# Pod status and events
kubectl get pods -l app=myapp -o wide
kubectl describe pod myapp-0

# Storage status
kubectl get pvc -l app=myapp
kubectl describe pvc data-myapp-0

# Service and DNS
kubectl get svc
nslookup myapp-0.myapp-headless.default.svc.cluster.local

# Scaling operations
kubectl get statefulset myapp -w
kubectl rollout status statefulset/myapp
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: When would you use StatefulSet vs Deployment?**
- **StatefulSet**: Stateful apps needing stable identity, persistent storage, ordered operations
- **Deployment**: Stateless apps that can be replaced/scaled randomly

**Q2: How does StatefulSet ensure ordered deployment?**
- Creates pods sequentially (0, 1, 2...)
- Waits for each pod to be Running and Ready before creating next
- Uses pod ordinal for naming and ordering

**Q3: What happens to PVCs when StatefulSet is deleted?**
- PVCs are NOT automatically deleted
- Data persists for manual recovery
- Must manually delete PVCs if no longer needed

### Technical Questions

**Q4: How do you perform zero-downtime updates on StatefulSet?**
```yaml
# Use partitioned rolling updates
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # Update gradually
```

**Q5: How does StatefulSet handle pod failures?**
- Failed pods are recreated with same name and storage
- Maintains stable identity and persistent data
- Follows podManagementPolicy for replacement

**Q6: What is a headless service and why is it needed?**
- Service with clusterIP: None
- Provides stable DNS names for each pod
- Enables direct pod-to-pod communication

### Scenario-Based Questions

**Q7: Design a highly available database cluster.**
```yaml
# Use StatefulSet with:
# - Anti-affinity for pod distribution
# - Persistent storage per pod
# - Health checks and monitoring
# - Backup and recovery procedures
# - Load balancer for read replicas
```

**Q8: How would you migrate data between StatefulSet pods?**
```bash
# 1. Scale down to single pod
# 2. Backup data from remaining pod
# 3. Update StatefulSet configuration
# 4. Scale up with new configuration
# 5. Restore data and verify
```

**Q9: Troubleshoot StatefulSet pods not starting.**
```bash
# Check pod events and logs
# Verify PVC and storage availability
# Check resource constraints
# Validate service account permissions
# Review init container failures
```

### Best Practices Questions

**Q10: What are StatefulSet best practices?**
- Use appropriate storage classes
- Implement proper health checks
- Plan for data backup and recovery
- Test scaling and update procedures
- Monitor storage usage and performance
- Use anti-affinity for high availability
- Implement proper security contexts
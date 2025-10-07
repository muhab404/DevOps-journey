# 🎯 Kubernetes Scenario-Based Interview Questions - Senior DevOps Guide

## Table of Contents
1. [Production Incidents](#production-incidents)
2. [Scaling & Performance](#scaling--performance)
3. [Security Scenarios](#security-scenarios)
4. [Networking Issues](#networking-issues)
5. [Storage Problems](#storage-problems)
6. [CI/CD Challenges](#cicd-challenges)
7. [Monitoring & Troubleshooting](#monitoring--troubleshooting)
8. [Multi-Cluster Management](#multi-cluster-management)
9. [Cost Optimization](#cost-optimization)
10. [Disaster Recovery](#disaster-recovery)

---

## 🔹 Production Incidents

### Scenario 1: Pod Crash Loop with Memory Issues

**Problem**: Your e-commerce application pods are crash looping during Black Friday traffic. Memory usage spikes and pods get OOMKilled.

**Investigation Steps**:
```bash
# Check pod status and events
kubectl get pods -l app=ecommerce
kubectl describe pod <pod-name>

# Check resource usage
kubectl top pods -l app=ecommerce
kubectl top nodes

# Check logs before crash
kubectl logs <pod-name> --previous

# Check HPA status
kubectl get hpa
kubectl describe hpa ecommerce-hpa
```

**Root Cause Analysis**:
```yaml
# Current problematic configuration
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"  # Too low for peak traffic

# Memory leak in application code
# Insufficient JVM heap settings
# No memory profiling
```

**Solution**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: ecommerce
  template:
    metadata:
      labels:
        app: ecommerce
    spec:
      containers:
      - name: app
        image: ecommerce:v2.1
        resources:
          requests:
            cpu: 500m
            memory: 1Gi      # Increased requests
          limits:
            cpu: 2000m
            memory: 4Gi      # Increased limits
        
        # JVM tuning for memory management
        env:
        - name: JAVA_OPTS
          value: "-Xms1g -Xmx3g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+HeapDumpOnOutOfMemoryError"
        
        # Liveness probe with longer timeout
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
---
# Aggressive HPA for traffic spikes
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-app
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

**Prevention Measures**:
```yaml
# Memory monitoring alert
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: memory-alerts
spec:
  groups:
  - name: memory.rules
    rules:
    - alert: HighMemoryUsage
      expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage detected"
        description: "Memory usage is above 80% for {{ $labels.pod }}"
```

---

### Scenario 2: Database Connection Pool Exhaustion

**Problem**: Your microservices are failing to connect to the database during peak hours. Connection timeouts and "too many connections" errors.

**Investigation**:
```bash
# Check database pod logs
kubectl logs -l app=postgres -f

# Check application logs
kubectl logs -l app=api-service | grep -i "connection"

# Check database connections
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"

# Check service endpoints
kubectl get endpoints postgres-service
```

**Root Cause**: 
- Default connection pool size too small
- No connection pooling at database level
- Long-running transactions not being closed

**Solution**:
```yaml
# PgBouncer connection pooler
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pgbouncer
  template:
    metadata:
      labels:
        app: pgbouncer
    spec:
      containers:
      - name: pgbouncer
        image: pgbouncer/pgbouncer:latest
        ports:
        - containerPort: 5432
        env:
        - name: DATABASES_HOST
          value: postgres-service
        - name: DATABASES_PORT
          value: "5432"
        - name: DATABASES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: DATABASES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POOL_MODE
          value: transaction
        - name: MAX_CLIENT_CONN
          value: "1000"
        - name: DEFAULT_POOL_SIZE
          value: "25"
        - name: MAX_DB_CONNECTIONS
          value: "100"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
# Updated application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://pgbouncer-service:5432/production
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5
          connection-timeout: 30000
          idle-timeout: 600000
          max-lifetime: 1800000
          leak-detection-threshold: 60000
```

---

## 🔹 Scaling & Performance

### Scenario 3: Sudden Traffic Spike - Auto-scaling Not Fast Enough

**Problem**: Your application receives 10x traffic during a viral social media mention. HPA is scaling but too slowly, causing 503 errors.

**Current Issue**:
```yaml
# Slow scaling HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: slow-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Default behavior - too conservative
```

**Solution - Multi-layered Scaling Strategy**:
```yaml
# 1. Aggressive HPA with custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: aggressive-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 5
  maxReplicas: 100
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 60
  
  # Custom metric - requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  
  # Queue depth metric
  - type: External
    external:
      metric:
        name: sqs_queue_depth
      target:
        type: AverageValue
        averageValue: "10"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30  # Fast scale up
      policies:
      - type: Percent
        value: 200  # Double pods quickly
        periodSeconds: 30
      - type: Pods
        value: 10   # Add 10 pods at once
        periodSeconds: 30
      selectPolicy: Max
    
    scaleDown:
      stabilizationWindowSeconds: 600  # Slow scale down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
---
# 2. Vertical Pod Autoscaler for right-sizing
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      maxAllowed:
        cpu: 2000m
        memory: 4Gi
      minAllowed:
        cpu: 100m
        memory: 128Mi
---
# 3. Cluster Autoscaler configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status
  namespace: kube-system
data:
  nodes.max: "100"
  scale-down-delay-after-add: "10m"
  scale-down-unneeded-time: "10m"
  skip-nodes-with-local-storage: "false"
  skip-nodes-with-system-pods: "false"
---
# 4. Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 50%
  selector:
    matchLabels:
      app: web-app
```

**Pre-scaling Strategy**:
```yaml
# Scheduled scaling for predictable traffic
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pre-scale-up
spec:
  schedule: "0 8 * * 1-5"  # Scale up weekdays at 8 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl patch hpa web-app-hpa -p '{"spec":{"minReplicas":20}}'
              echo "Scaled up for business hours"
          restartPolicy: OnFailure
---
# Scale down after hours
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pre-scale-down
spec:
  schedule: "0 20 * * 1-5"  # Scale down weekdays at 8 PM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl patch hpa web-app-hpa -p '{"spec":{"minReplicas":5}}'
              echo "Scaled down for off hours"
          restartPolicy: OnFailure
```

---

## 🔹 Security Scenarios

### Scenario 4: Security Breach - Compromised Container

**Problem**: Security team alerts that one of your pods is making suspicious outbound connections to unknown IPs. Potential container compromise.

**Immediate Response**:
```bash
# 1. Identify the compromised pod
kubectl get pods -o wide | grep <suspicious-ip>

# 2. Isolate the pod immediately
kubectl label pod <pod-name> quarantine=true

# 3. Apply emergency network policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine-policy
spec:
  podSelector:
    matchLabels:
      quarantine: "true"
  policyTypes:
  - Ingress
  - Egress
  egress: []  # Block all egress traffic
  ingress: [] # Block all ingress traffic
EOF

# 4. Collect forensic data
kubectl logs <pod-name> > compromised-pod-logs.txt
kubectl describe pod <pod-name> > pod-description.txt
kubectl exec <pod-name> -- ps aux > running-processes.txt
kubectl exec <pod-name> -- netstat -tulpn > network-connections.txt
```

**Investigation and Remediation**:
```yaml
# 1. Enhanced security deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
      annotations:
        # Security scanning
        container.apparmor.security.beta.kubernetes.io/app: runtime/default
    spec:
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: app
        image: myapp:v1.2-secure
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop: ["ALL"]
            add: ["NET_BIND_SERVICE"]  # Only if needed
          seccompProfile:
            type: RuntimeDefault
        
        # Resource limits to prevent resource exhaustion attacks
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # Minimal writable volumes
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: var-cache
          mountPath: /var/cache/app
        
        # Health checks
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
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: var-cache
        emptyDir: {}
---
# 2. Strict network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: secure-app-policy
spec:
  podSelector:
    matchLabels:
      app: secure-app
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Only allow ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  # DNS only
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  
  # Database access only
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  
  # No external internet access
---
# 3. Pod Security Policy (or Pod Security Standards)
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Monitoring and Alerting**:
```yaml
# Falco rules for runtime security
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-rules
data:
  custom_rules.yaml: |
    - rule: Suspicious Network Activity
      desc: Detect suspicious outbound connections
      condition: >
        outbound and fd.sip not in (allowed_ips) and
        not proc.name in (allowed_processes)
      output: >
        Suspicious outbound connection
        (user=%user.name command=%proc.cmdline connection=%fd.name)
      priority: HIGH
      tags: [network, mitre_exfiltration]
    
    - rule: Unexpected Process Execution
      desc: Detect unexpected process execution in containers
      condition: >
        spawned_process and container and
        not proc.name in (allowed_processes) and
        not proc.pname in (allowed_parent_processes)
      output: >
        Unexpected process spawned in container
        (user=%user.name command=%proc.cmdline container=%container.name)
      priority: HIGH
      tags: [process, mitre_execution]
```

---

## 🔹 Networking Issues

### Scenario 5: Intermittent Service Communication Failures

**Problem**: Microservice A can sometimes connect to Microservice B, but connections fail randomly with DNS resolution errors and timeouts.

**Investigation**:
```bash
# Check DNS resolution
kubectl exec -it <pod-name> -- nslookup service-b.namespace.svc.cluster.local

# Check service endpoints
kubectl get endpoints service-b
kubectl describe service service-b

# Check network policies
kubectl get networkpolicies
kubectl describe networkpolicy <policy-name>

# Test connectivity
kubectl exec -it <pod-name> -- curl -v http://service-b:8080/health

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Root Causes Found**:
1. CoreDNS intermittent failures
2. Service endpoints not updating properly
3. Network policy blocking some traffic
4. Load balancer health check issues

**Solution**:
```yaml
# 1. Fix CoreDNS configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
---
# 2. Improved service configuration
apiVersion: v1
kind: Service
metadata:
  name: service-b
  labels:
    app: service-b
spec:
  selector:
    app: service-b
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
  
  # Session affinity for sticky sessions if needed
  sessionAffinity: None
  
  # External traffic policy
  externalTrafficPolicy: Cluster
  
  # Service topology (if using topology aware routing)
  topologyKeys:
  - "kubernetes.io/hostname"
  - "topology.kubernetes.io/zone"
  - "*"
---
# 3. Deployment with proper health checks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
spec:
  replicas: 3
  selector:
    matchLabels:
      app: service-b
  template:
    metadata:
      labels:
        app: service-b
    spec:
      containers:
      - name: service-b
        image: service-b:v1.0
        ports:
        - containerPort: 8080
        
        # Proper health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 1
        
        # Startup probe for slow-starting services
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
# 4. Network policy allowing communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-service-communication
spec:
  podSelector:
    matchLabels:
      app: service-b
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: service-a
    ports:
    - protocol: TCP
      port: 8080
---
# 5. Service mesh for better observability (Istio)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: service-b-dr
spec:
  host: service-b
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30s
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 3
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

**Client-side Resilience**:
```yaml
# Service A with retry logic
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-a-config
data:
  application.yml: |
    resilience4j:
      retry:
        instances:
          service-b:
            maxAttempts: 3
            waitDuration: 1s
            exponentialBackoffMultiplier: 2
      circuitbreaker:
        instances:
          service-b:
            failureRateThreshold: 50
            waitDurationInOpenState: 30s
            slidingWindowSize: 10
      timeout:
        instances:
          service-b:
            timeoutDuration: 5s
```

---

## 🔹 Storage Problems

### Scenario 6: Persistent Volume Claims Stuck in Pending

**Problem**: New PVCs are stuck in "Pending" state, preventing StatefulSet pods from starting. Storage is running out.

**Investigation**:
```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc <pvc-name>

# Check available storage
kubectl get pv
kubectl get storageclass

# Check node storage
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check storage provisioner logs
kubectl logs -n kube-system -l app=gce-pd-csi-driver
```

**Root Causes**:
1. Storage quota exceeded
2. No available nodes with sufficient storage
3. Storage class misconfiguration
4. Zone constraints not met

**Solution**:
```yaml
# 1. Create optimized storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd-regional
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zones: us-central1-a,us-central1-b,us-central1-c
  replication-type: regional-pd
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
---
# 2. StatefulSet with proper volume management
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - database
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: database
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
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        
        ports:
        - containerPort: 5432
        
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
  
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd-regional
      resources:
        requests:
          storage: 100Gi
---
# 3. Storage monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-alerts
spec:
  groups:
  - name: storage.rules
    rules:
    - alert: PVCPendingTooLong
      expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "PVC pending for too long"
        description: "PVC {{ $labels.persistentvolumeclaim }} has been pending for more than 5 minutes"
    
    - alert: StorageSpaceLow
      expr: (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Storage space running low"
        description: "Volume {{ $labels.persistentvolumeclaim }} is {{ $value | humanizePercentage }} full"
---
# 4. Automated cleanup job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pv-cleanup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Clean up Released PVs older than 7 days
              kubectl get pv -o json | jq -r '.items[] | select(.status.phase=="Released") | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 604800)) | .metadata.name' | xargs -r kubectl delete pv
              
              # Clean up Failed PVCs
              kubectl get pvc --all-namespaces -o json | jq -r '.items[] | select(.status.phase=="Failed") | "\(.metadata.namespace) \(.metadata.name)"' | while read ns name; do kubectl delete pvc -n $ns $name; done
          restartPolicy: OnFailure
```

---

## 🔹 CI/CD Challenges

### Scenario 7: Deployment Pipeline Failures

**Problem**: Your GitOps pipeline with ArgoCD is failing. Some applications are out of sync, rollbacks aren't working, and developers are blocked from deploying.

**Investigation**:
```bash
# Check ArgoCD application status
kubectl get applications -n argocd
kubectl describe application web-app -n argocd

# Check ArgoCD server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server

# Check repository connectivity
kubectl exec -n argocd argocd-server-xxx -- argocd repo list

# Check sync status
kubectl exec -n argocd argocd-server-xxx -- argocd app get web-app
```

**Root Causes**:
1. Git repository authentication issues
2. Kubernetes RBAC permissions
3. Resource conflicts
4. Image pull failures

**Solution**:
```yaml
# 1. Fix ArgoCD RBAC
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-application-controller
  namespace: argocd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-application-controller
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- nonResourceURLs: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-application-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-application-controller
subjects:
- kind: ServiceAccount
  name: argocd-application-controller
  namespace: argocd
---
# 2. Improved Application configuration
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-manifests
    targetRevision: HEAD
    path: apps/web-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    - RespectIgnoreDifferences=true
    - ApplyOutOfSyncOnly=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Ignore differences for certain fields
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
  - group: ""
    kind: Service
    jsonPointers:
    - /spec/clusterIP
---
# 3. Multi-stage deployment with canary
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web-app-rollout
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 30s}
      - setWeight: 20
      - pause: {duration: 30s}
      - setWeight: 50
      - pause: {duration: 60s}
      - setWeight: 100
      
      canaryService: web-app-canary
      stableService: web-app-stable
      
      analysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: web-app-canary
        
        startingStep: 2
        
      trafficRouting:
        nginx:
          stableIngress: web-app-ingress
          annotationPrefix: nginx.ingress.kubernetes.io
          additionalIngressAnnotations:
            canary-by-header: X-Canary
  
  selector:
    matchLabels:
      app: web-app
  
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: web-app:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
# 4. Analysis template for automated rollback
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 30s
    count: 5
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[5m])) /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
  
  - name: avg-response-time
    interval: 30s
    count: 5
    successCondition: result[0] <= 0.5
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket{service="{{args.service-name}}"}[5m])) by (le)
          )
```

**GitHub Actions Integration**:
```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Kustomize
      uses: imranismail/setup-kustomize@v1
    
    - name: Update image tag
      run: |
        cd k8s/overlays/production
        kustomize edit set image web-app=web-app:${{ github.sha }}
        kustomize build . > ../../../manifests.yaml
    
    - name: Commit changes
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"
        git add manifests.yaml
        git commit -m "Update image to ${{ github.sha }}" || exit 0
        git push
    
    - name: Trigger ArgoCD sync
      run: |
        # Wait for ArgoCD to detect changes
        sleep 30
        
        # Force sync if needed
        kubectl exec -n argocd deployment/argocd-server -- \
          argocd app sync web-app --force
```

---

## 🔹 Monitoring & Troubleshooting

### Scenario 8: Application Performance Degradation

**Problem**: Your application response times have increased from 100ms to 2000ms over the past week. Users are complaining about slow performance.

**Investigation Strategy**:
```bash
# Check application metrics
kubectl top pods -l app=web-app
kubectl get hpa web-app-hpa

# Check resource utilization
kubectl describe nodes | grep -A 10 "Allocated resources"

# Check application logs for errors
kubectl logs -l app=web-app --tail=1000 | grep -i error

# Check database performance
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT query, mean_time, calls FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;"
```

**Comprehensive Monitoring Setup**:
```yaml
# 1. Application Performance Monitoring
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-monitored
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web-app
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
      - name: web-app
        image: web-app:v1.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        
        env:
        # Enable detailed metrics
        - name: METRICS_ENABLED
          value: "true"
        - name: TRACING_ENABLED
          value: "true"
        - name: JAEGER_ENDPOINT
          value: "http://jaeger-collector:14268/api/traces"
        
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
      
      # Sidecar for additional monitoring
      - name: jaeger-agent
        image: jaegertracing/jaeger-agent:latest
        ports:
        - containerPort: 5775
          protocol: UDP
        - containerPort: 6831
          protocol: UDP
        - containerPort: 6832
          protocol: UDP
        - containerPort: 5778
          protocol: TCP
        args:
        - --collector.host-port=jaeger-collector:14267
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
---
# 2. Comprehensive alerting rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: performance-alerts
spec:
  groups:
  - name: performance.rules
    rules:
    # Response time alert
    - alert: HighResponseTime
      expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="web-app"}[5m])) by (le)) > 1.0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High response time detected"
        description: "95th percentile response time is {{ $value }}s"
    
    # Error rate alert
    - alert: HighErrorRate
      expr: sum(rate(http_requests_total{job="web-app",status=~"5.."}[5m])) / sum(rate(http_requests_total{job="web-app"}[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }}"
    
    # Database connection pool alert
    - alert: DatabaseConnectionPoolExhaustion
      expr: db_connection_pool_active / db_connection_pool_max > 0.8
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Database connection pool nearly exhausted"
        description: "Connection pool is {{ $value | humanizePercentage }} full"
    
    # Memory leak detection
    - alert: MemoryLeakDetected
      expr: increase(container_memory_usage_bytes{pod=~"web-app-.*"}[1h]) > 100000000  # 100MB increase per hour
      for: 2h
      labels:
        severity: warning
      annotations:
        summary: "Potential memory leak detected"
        description: "Memory usage increased by {{ $value | humanizeBytes }} in the last hour"
---
# 3. Performance testing job
apiVersion: batch/v1
kind: Job
metadata:
  name: performance-test
spec:
  template:
    spec:
      containers:
      - name: k6
        image: loadimpact/k6:latest
        command: ["k6", "run", "--vus", "50", "--duration", "5m", "/scripts/load-test.js"]
        volumeMounts:
        - name: test-scripts
          mountPath: /scripts
        env:
        - name: TARGET_URL
          value: "http://web-app-service:8080"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
      
      volumes:
      - name: test-scripts
        configMap:
          name: performance-test-scripts
      
      restartPolicy: Never
---
# 4. Load test script
apiVersion: v1
kind: ConfigMap
metadata:
  name: performance-test-scripts
data:
  load-test.js: |
    import http from 'k6/http';
    import { check, sleep } from 'k6';
    
    export let options = {
      stages: [
        { duration: '1m', target: 10 },
        { duration: '2m', target: 50 },
        { duration: '1m', target: 100 },
        { duration: '1m', target: 0 },
      ],
      thresholds: {
        http_req_duration: ['p(95)<500'],
        http_req_failed: ['rate<0.05'],
      },
    };
    
    export default function() {
      let response = http.get(`${__ENV.TARGET_URL}/api/users`);
      check(response, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
      });
      sleep(1);
    }
```

**Performance Optimization**:
```yaml
# Database optimization
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    # Memory settings
    shared_buffers = 256MB
    effective_cache_size = 1GB
    work_mem = 4MB
    maintenance_work_mem = 64MB
    
    # Connection settings
    max_connections = 200
    
    # Performance settings
    random_page_cost = 1.1
    effective_io_concurrency = 200
    
    # Logging
    log_min_duration_statement = 1000
    log_statement = 'all'
    log_duration = on
---
# Application caching layer
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:6-alpine
        ports:
        - containerPort: 6379
        command: ["redis-server"]
        args: ["--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lru"]
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

---

## 🔹 Cost Optimization

### Scenario 9: Cloud Costs Spiraling Out of Control

**Problem**: Your Kubernetes cluster costs have tripled in the last month. Management wants a 40% cost reduction without impacting performance.

**Cost Analysis**:
```bash
# Check resource utilization
kubectl top nodes
kubectl top pods --all-namespaces

# Check unused resources
kubectl get pv | grep Available
kubectl get pvc --all-namespaces | grep Pending

# Analyze resource requests vs usage
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**Cost Optimization Strategy**:
```yaml
# 1. Right-sizing with VPA
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: cost-optimized-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      maxAllowed:
        cpu: 1000m
        memory: 2Gi
      minAllowed:
        cpu: 50m
        memory: 64Mi
      controlledResources: ["cpu", "memory"]
---
# 2. Spot instance node pool
apiVersion: v1
kind: ConfigMap
metadata:
  name: spot-instances-config
data:
  create-spot-pool.sh: |
    #!/bin/bash
    gcloud container node-pools create spot-pool \
        --cluster=production-cluster \
        --zone=us-central1-a \
        --machine-type=e2-standard-4 \
        --spot \
        --num-nodes=0 \
        --enable-autoscaling \
        --min-nodes=0 \
        --max-nodes=20 \
        --node-taints=cloud.google.com/gke-spot=true:NoSchedule \
        --node-labels=cloud.google.com/gke-spot=true,cost-optimized=true
---
# 3. Cost-optimized deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor-spot
spec:
  replicas: 5
  selector:
    matchLabels:
      app: batch-processor
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      # Tolerate spot instance interruptions
      tolerations:
      - key: cloud.google.com/gke-spot
        operator: Equal
        value: "true"
        effect: NoSchedule
      
      # Prefer spot instances
      nodeSelector:
        cloud.google.com/gke-spot: "true"
      
      # Graceful shutdown handling
      terminationGracePeriodSeconds: 30
      
      containers:
      - name: processor
        image: batch-processor:v1.0
        
        # Right-sized resources
        resources:
          requests:
            cpu: 100m      # Reduced from 500m
            memory: 128Mi  # Reduced from 512Mi
          limits:
            cpu: 200m      # Reduced from 1000m
            memory: 256Mi  # Reduced from 1Gi
        
        # Handle spot interruptions gracefully
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        
        env:
        - name: GRACEFUL_SHUTDOWN_TIMEOUT
          value: "25"
---
# 4. Resource quotas to prevent waste
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cost-control-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    persistentvolumeclaims: "20"
    services.loadbalancers: "2"
---
# 5. Automated resource cleanup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: resource-cleanup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Clean up completed jobs older than 24 hours
              kubectl get jobs --all-namespaces -o json | \
                jq -r '.items[] | select(.status.conditions[]?.type=="Complete") | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 86400)) | "\(.metadata.namespace) \(.metadata.name)"' | \
                while read ns name; do kubectl delete job -n $ns $name; done
              
              # Clean up failed pods older than 1 hour
              kubectl get pods --all-namespaces --field-selector=status.phase=Failed -o json | \
                jq -r '.items[] | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 3600)) | "\(.metadata.namespace) \(.metadata.name)"' | \
                while read ns name; do kubectl delete pod -n $ns $name; done
              
              # Clean up unused ConfigMaps and Secrets
              # (Add logic based on your specific needs)
          restartPolicy: OnFailure
---
# 6. Cost monitoring alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cost-alerts
spec:
  groups:
  - name: cost.rules
    rules:
    - alert: HighResourceWaste
      expr: (kube_pod_container_resource_requests{resource="cpu"} - rate(container_cpu_usage_seconds_total[5m])) / kube_pod_container_resource_requests{resource="cpu"} > 0.5
      for: 30m
      labels:
        severity: warning
      annotations:
        summary: "High CPU resource waste detected"
        description: "Pod {{ $labels.pod }} is wasting {{ $value | humanizePercentage }} of requested CPU"
    
    - alert: UnusedPersistentVolumes
      expr: kube_persistentvolume_status_phase{phase="Available"} == 1
      for: 24h
      labels:
        severity: info
      annotations:
        summary: "Unused persistent volume detected"
        description: "PV {{ $labels.persistentvolume }} has been available for more than 24 hours"
```

**Cost Optimization Strategies**:
```yaml
# Cluster autoscaler optimization
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-config
  namespace: kube-system
data:
  # Aggressive scale-down for cost savings
  scale-down-delay-after-add: "5m"
  scale-down-unneeded-time: "5m"
  scale-down-utilization-threshold: "0.5"
  skip-nodes-with-local-storage: "false"
  skip-nodes-with-system-pods: "false"
  max-node-provision-time: "10m"
---
# Mixed instance types for cost optimization
apiVersion: v1
kind: ConfigMap
metadata:
  name: mixed-instance-nodepool
data:
  create-mixed-pool.sh: |
    #!/bin/bash
    # Create node pool with mixed instance types
    gcloud container node-pools create mixed-pool \
        --cluster=production-cluster \
        --zone=us-central1-a \
        --num-nodes=0 \
        --enable-autoscaling \
        --min-nodes=0 \
        --max-nodes=15 \
        --machine-type=e2-standard-2 \
        --spot \
        --node-labels=cost-tier=low \
        --preemptible
```

This comprehensive scenario-based guide covers real-world Kubernetes challenges that senior DevOps engineers face in production environments. Each scenario includes detailed investigation steps, root cause analysis, and complete solutions with monitoring and prevention strategies.
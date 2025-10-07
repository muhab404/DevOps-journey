# 🎯 Advanced Kubernetes Scenario Questions - Senior DevOps Guide

## Table of Contents
1. [Production Crisis Management](#production-crisis-management)
2. [Performance & Scaling Challenges](#performance--scaling-challenges)
3. [Security Incidents](#security-incidents)
4. [Networking Complexities](#networking-complexities)
5. [Storage & Data Management](#storage--data-management)
6. [Multi-Cluster Operations](#multi-cluster-operations)
7. [CI/CD Pipeline Issues](#cicd-pipeline-issues)
8. [Resource Management](#resource-management)
9. [Observability & Debugging](#observability--debugging)
10. [Architecture & Design Decisions](#architecture--design-decisions)

---

## 🔹 Production Crisis Management

### Scenario 1: Complete Application Outage During Peak Traffic

**Situation**: Your e-commerce platform serving 1M+ users goes completely down during Black Friday. All pods are in CrashLoopBackOff state, and customers can't access the site.

**Initial Assessment**:
```bash
# Quick health check
kubectl get pods --all-namespaces | grep -v Running
kubectl get nodes
kubectl top nodes
kubectl get events --sort-by=.metadata.creationTimestamp | tail -20
```

**Investigation Process**:
```bash
# 1. Check cluster-level issues
kubectl cluster-info
kubectl get componentstatuses

# 2. Analyze failing pods
kubectl describe pod <failing-pod> -n production
kubectl logs <failing-pod> -n production --previous

# 3. Check resource constraints
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl get limitrange --all-namespaces
kubectl get resourcequota --all-namespaces

# 4. Examine recent changes
kubectl rollout history deployment/web-app -n production
kubectl get events --sort-by=.metadata.creationTimestamp | grep -i error
```

**Root Cause Discovery**:
```yaml
# Found: Memory limits too restrictive after recent deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          limits:
            memory: "128Mi"  # Too low for Black Friday traffic
          requests:
            memory: "64Mi"
```

**Immediate Recovery Actions**:
```bash
# 1. Emergency resource increase
kubectl patch deployment web-app -n production -p='{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"limits":{"memory":"2Gi"},"requests":{"memory":"1Gi"}}}]}}}}'

# 2. Scale up replicas
kubectl scale deployment web-app -n production --replicas=20

# 3. Check rollout status
kubectl rollout status deployment/web-app -n production

# 4. Verify recovery
kubectl get pods -n production -l app=web-app
curl -f http://your-app-url/health
```

**Long-term Solution**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: web-app:v2.1
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        
        # Proper health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Startup probe for slow initialization
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 30
---
# Aggressive HPA for traffic spikes
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 10
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 60
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
# Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
spec:
  minAvailable: 70%
  selector:
    matchLabels:
      app: web-app
---
# Resource monitoring alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: critical-alerts
  namespace: monitoring
spec:
  groups:
  - name: critical.rules
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
    
    - alert: DeploymentReplicasMismatch
      expr: kube_deployment_spec_replicas != kube_deployment_status_available_replicas
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Deployment {{ $labels.deployment }} has mismatched replicas"
```

---

### Scenario 2: Database Connection Pool Exhaustion

**Situation**: Your microservices architecture is failing with "too many connections" errors. Database connections are maxed out, causing cascading failures across services.

**Investigation**:
```bash
# Check application logs
kubectl logs -l app=api-service -n production | grep -i "connection"

# Check database pod
kubectl exec -it postgres-0 -n production -- psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"

# Check connection pool configuration
kubectl get configmap app-config -n production -o yaml

# Monitor connection patterns
kubectl top pods -n production --sort-by=memory
```

**Root Cause Analysis**:
```yaml
# Problem: Each pod creating too many connections
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.properties: |
    spring.datasource.hikari.maximum-pool-size=50  # Too high per pod
    spring.datasource.hikari.minimum-idle=10
```

**Solution - Connection Pooling with PgBouncer**:
```yaml
# PgBouncer Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: production
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
        livenessProbe:
          tcpSocket:
            port: 5432
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          tcpSocket:
            port: 5432
          initialDelaySeconds: 5
          periodSeconds: 5
---
# PgBouncer Service
apiVersion: v1
kind: Service
metadata:
  name: pgbouncer-service
  namespace: production
spec:
  selector:
    app: pgbouncer
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
---
# Updated application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
  namespace: production
data:
  database.properties: |
    spring.datasource.url=jdbc:postgresql://pgbouncer-service:5432/production
    spring.datasource.hikari.maximum-pool-size=10  # Reduced per pod
    spring.datasource.hikari.minimum-idle=2
    spring.datasource.hikari.connection-timeout=30000
    spring.datasource.hikari.idle-timeout=600000
    spring.datasource.hikari.max-lifetime=1800000
    spring.datasource.hikari.leak-detection-threshold=60000
```

**Circuit Breaker Implementation**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: resilience-config
  namespace: production
data:
  application.yml: |
    resilience4j:
      circuitbreaker:
        instances:
          database:
            failureRateThreshold: 50
            waitDurationInOpenState: 30s
            slidingWindowSize: 10
            minimumNumberOfCalls: 5
      retry:
        instances:
          database:
            maxAttempts: 3
            waitDuration: 1s
            exponentialBackoffMultiplier: 2
      timeout:
        instances:
          database:
            timeoutDuration: 5s
```

---

## 🔹 Performance & Scaling Challenges

### Scenario 3: Microservices Performance Degradation

**Situation**: Your microservices architecture is experiencing 10x increase in response times. Some services are timing out, and the system is becoming unresponsive.

**Performance Investigation**:
```bash
# Check service response times
kubectl exec -it debug-pod -- curl -w "@curl-format.txt" -o /dev/null -s http://user-service:8080/users/1

# Monitor resource usage
kubectl top pods --sort-by=cpu -n production
kubectl top pods --sort-by=memory -n production

# Check service mesh metrics (if using Istio)
kubectl exec -it istio-proxy -c istio-proxy -- pilot-agent request GET stats/prometheus | grep histogram

# Analyze application metrics
kubectl port-forward svc/prometheus 9090:9090
# Query: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
```

**Root Cause Analysis**:
```bash
# Check for resource constraints
kubectl describe nodes | grep -A 10 "Allocated resources"

# Analyze slow database queries
kubectl logs -l app=user-service | grep "slow query"

# Check for memory leaks
kubectl exec -it user-service-pod -- jstat -gc 1 5s

# Network latency between services
kubectl exec -it debug-pod -- traceroute user-service
```

**Solution - Performance Optimization**:
```yaml
# Optimized deployment with proper resource allocation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: user-service
        image: user-service:v2.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        
        # Optimized resource allocation
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        
        # JVM optimization
        env:
        - name: JAVA_OPTS
          value: "-Xms1g -Xmx3g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+UseStringDeduplication"
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        
        # Health checks with appropriate timeouts
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Startup probe for slow-starting services
        startupProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 30
      
      # Pod anti-affinity for better distribution
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
                  - user-service
              topologyKey: kubernetes.io/hostname
---
# Aggressive HPA with custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
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
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
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

**Caching Layer Implementation**:
```yaml
# Redis Cache Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
  namespace: production
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
        args: 
        - "--maxmemory"
        - "1gb"
        - "--maxmemory-policy"
        - "allkeys-lru"
        - "--save"
        - ""
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Application configuration with caching
apiVersion: v1
kind: ConfigMap
metadata:
  name: cache-config
  namespace: production
data:
  application.yml: |
    spring:
      cache:
        type: redis
      redis:
        host: redis-service
        port: 6379
        timeout: 2000ms
        lettuce:
          pool:
            max-active: 20
            max-idle: 10
            min-idle: 5
    
    # Cache configuration
    cache:
      user-cache:
        ttl: 300  # 5 minutes
      session-cache:
        ttl: 1800  # 30 minutes
```

---

## 🔹 Security Incidents

### Scenario 4: Container Security Breach

**Situation**: Security team alerts that one of your pods is making suspicious outbound connections and may be compromised. You need to contain the threat and investigate.

**Immediate Response**:
```bash
# 1. Identify the suspicious pod
kubectl get pods -o wide | grep <suspicious-ip>

# 2. Isolate the pod immediately
kubectl label pod <suspicious-pod> security.breach=quarantine

# 3. Apply emergency network policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      security.breach: quarantine
  policyTypes:
  - Ingress
  - Egress
  egress: []  # Block all egress
  ingress: [] # Block all ingress
EOF

# 4. Collect forensic evidence
kubectl logs <suspicious-pod> > breach-logs-$(date +%s).txt
kubectl describe pod <suspicious-pod> > pod-description-$(date +%s).txt
kubectl exec <suspicious-pod> -- ps aux > running-processes-$(date +%s).txt
kubectl exec <suspicious-pod> -- netstat -tulpn > network-connections-$(date +%s).txt
```

**Investigation Process**:
```bash
# Check for privilege escalation
kubectl exec <suspicious-pod> -- cat /proc/1/status | grep -i cap

# Look for unauthorized file modifications
kubectl exec <suspicious-pod> -- find /usr /bin /sbin -newer /proc/1/stat 2>/dev/null

# Check for crypto mining processes
kubectl exec <suspicious-pod> -- ps aux | grep -E "(xmrig|minerd|cpuminer)"

# Analyze network connections
kubectl exec <suspicious-pod> -- ss -tuln | grep LISTEN

# Check for backdoors
kubectl exec <suspicious-pod> -- find /tmp /var/tmp -type f -executable
```

**Remediation and Hardening**:
```yaml
# Secure replacement deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app-v2
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
      version: v2
  template:
    metadata:
      labels:
        app: secure-app
        version: v2
      annotations:
        container.apparmor.security.beta.kubernetes.io/app: runtime/default
    spec:
      # Security context at pod level
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: app
        image: secure-app:v2.1-hardened
        securityContext:
          # Container-level security
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop: ["ALL"]
            add: ["NET_BIND_SERVICE"]  # Only if needed
          seccompProfile:
            type: RuntimeDefault
        
        # Resource limits to prevent resource exhaustion
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
# Strict network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: secure-app-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: secure-app
      version: v2
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
```

**Runtime Security Monitoring**:
```yaml
# Falco rules for runtime security
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-rules
  namespace: falco-system
data:
  custom_rules.yaml: |
    - rule: Suspicious Network Activity
      desc: Detect suspicious outbound connections
      condition: >
        outbound and fd.sip not in (allowed_ips) and
        not proc.name in (allowed_processes)
      output: >
        Suspicious outbound connection
        (user=%user.name command=%proc.cmdline connection=%fd.name
        container=%container.name image=%container.image.repository)
      priority: HIGH
      tags: [network, mitre_exfiltration]
    
    - rule: Unexpected Process Execution
      desc: Detect unexpected process execution
      condition: >
        spawned_process and container and
        not proc.name in (java, node, python, nginx, redis-server) and
        not proc.pname in (java, node, python, nginx, redis-server)
      output: >
        Unexpected process spawned in container
        (user=%user.name command=%proc.cmdline container=%container.name
        image=%container.image.repository parent=%proc.pname)
      priority: HIGH
      tags: [process, mitre_execution]
    
    - rule: File System Modification
      desc: Detect unauthorized file modifications
      condition: >
        open_write and container and
        fd.name startswith /usr or fd.name startswith /bin or
        fd.name startswith /sbin or fd.name startswith /etc
      output: >
        Unauthorized file modification
        (user=%user.name file=%fd.name command=%proc.cmdline
        container=%container.name)
      priority: HIGH
      tags: [filesystem, mitre_persistence]
```

---

## 🔹 Networking Complexities

### Scenario 5: Service Mesh Traffic Management Crisis

**Situation**: After implementing Istio service mesh, you're experiencing intermittent 503 errors, traffic routing issues, and some services are completely unreachable.

**Investigation Process**:
```bash
# Check Istio control plane
kubectl get pods -n istio-system
kubectl logs -n istio-system -l app=istiod

# Verify sidecar injection
kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].name}' | grep istio-proxy

# Check service mesh configuration
kubectl get virtualservices -n production
kubectl get destinationrules -n production
kubectl get gateways -n production

# Analyze traffic patterns
kubectl exec -it <pod-with-istio-proxy> -c istio-proxy -- pilot-agent request GET stats/prometheus | grep -E "(upstream_rq_|downstream_rq_)"
```

**Root Cause Analysis**:
```bash
# Check for configuration conflicts
istioctl analyze -n production

# Verify proxy configuration
istioctl proxy-config cluster <pod-name> -n production
istioctl proxy-config listener <pod-name> -n production
istioctl proxy-config route <pod-name> -n production

# Check certificate issues
istioctl proxy-config secret <pod-name> -n production
```

**Solution - Proper Service Mesh Configuration**:
```yaml
# Gateway configuration
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: production-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - api.example.com
    tls:
      httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - api.example.com
    tls:
      mode: SIMPLE
      credentialName: api-tls-secret
---
# Virtual Service with proper routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-routing
  namespace: production
spec:
  hosts:
  - api.example.com
  gateways:
  - production-gateway
  http:
  # Canary routing
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: api-service
        subset: canary
      weight: 100
    fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
  
  # Default routing with traffic split
  - route:
    - destination:
        host: api-service
        subset: stable
      weight: 90
    - destination:
        host: api-service
        subset: canary
      weight: 10
    
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
      retryOn: 5xx,reset,connect-failure,refused-stream
---
# Destination Rule with circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-service-dr
  namespace: production
spec:
  host: api-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30s
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        http1MaxPendingRequests: 10
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
        maxRetries: 3
        consecutiveGatewayErrors: 5
        interval: 30s
        baseEjectionTime: 30s
        maxEjectionPercent: 50
    
    outlierDetection:
      consecutiveGatewayErrors: 3
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
  
  subsets:
  - name: stable
    labels:
      version: stable
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  
  - name: canary
    labels:
      version: canary
    trafficPolicy:
      loadBalancer:
        simple: LEAST_CONN
---
# Service Entry for external dependencies
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
  namespace: production
spec:
  hosts:
  - external-api.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend-sa"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/v1/*"]
    when:
    - key: request.headers[authorization]
      values: ["Bearer *"]
```

**Traffic Management Debugging**:
```bash
# Enable debug logging
kubectl patch configmap istio -n istio-system --type merge -p='{"data":{"mesh":"defaultConfig:\n  proxyStatsMatcher:\n    inclusionRegexps:\n    - \".*circuit_breakers.*\"\n    - \".*upstream_rq_retry.*\"\n    - \".*upstream_rq_pending.*\"\n    - \".*_cx_.*\""}}'

# Check traffic distribution
kubectl exec -it <pod> -c istio-proxy -- pilot-agent request GET stats/prometheus | grep -E "upstream_rq_total|upstream_rq_pending"

# Analyze circuit breaker status
kubectl exec -it <pod> -c istio-proxy -- pilot-agent request GET stats/prometheus | grep circuit_breakers

# Test connectivity
kubectl exec -it debug-pod -- curl -v http://api-service:8080/health
```

This comprehensive guide covers advanced Kubernetes scenarios that senior DevOps engineers encounter in production environments, with detailed investigation processes, root cause analysis, and complete solutions.
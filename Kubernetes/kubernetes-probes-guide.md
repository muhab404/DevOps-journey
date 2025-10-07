# 🔍 Kubernetes Probes - DevOps Interview Guide

## Table of Contents
1. [Probes Overview](#probes-overview)
2. [Liveness Probes](#liveness-probes)
3. [Readiness Probes](#readiness-probes)
4. [Startup Probes](#startup-probes)
5. [Probe Types](#probe-types)
6. [Probe Configuration](#probe-configuration)
7. [Practical Examples](#practical-examples)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)
10. [Interview Questions](#interview-questions)

---

## 🔹 Probes Overview

### What are Kubernetes Probes?

**Probes** are health checks that Kubernetes uses to determine the state of containers and make decisions about pod lifecycle management.

### Types of Probes

| Probe Type | Purpose | Action on Failure |
|------------|---------|-------------------|
| **Liveness** | Check if container is running | Restart container |
| **Readiness** | Check if container is ready to serve traffic | Remove from service endpoints |
| **Startup** | Check if container has started | Wait before other probes |

### Probe Execution Flow

```yaml
# Probe Execution Order:
1. Startup Probe (if configured) - runs first
2. Liveness Probe - runs throughout container lifecycle
3. Readiness Probe - runs throughout container lifecycle

# Probe States:
Success: Probe passed
Failure: Probe failed
Unknown: Probe execution failed (network issues, etc.)
```

---

## 🔹 Liveness Probes

### Purpose and Behavior

**Liveness Probes** determine if a container is running properly. If a liveness probe fails, Kubernetes kills the container and restarts it according to the restart policy.

### Basic Liveness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-example
spec:
  containers:
  - name: app
    image: nginx:1.21
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 30    # Wait 30s before first probe
      periodSeconds: 10          # Probe every 10s
      timeoutSeconds: 5          # Timeout after 5s
      failureThreshold: 3        # Fail after 3 consecutive failures
      successThreshold: 1        # Success after 1 success (always 1 for liveness)
```

### Liveness Probe Use Cases

```yaml
# When to use Liveness Probes:
- Detect application deadlocks
- Restart hung processes
- Handle memory leaks
- Recover from infinite loops
- Restart after critical errors

# When NOT to use:
- Temporary failures (use readiness instead)
- External dependency failures
- Startup delays (use startup probe)
```

---

## 🔹 Readiness Probes

### Purpose and Behavior

**Readiness Probes** determine if a container is ready to serve requests. If a readiness probe fails, Kubernetes removes the pod from service endpoints but doesn't restart the container.

### Basic Readiness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-example
spec:
  containers:
  - name: app
    image: nginx:1.21
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5     # Wait 5s before first probe
      periodSeconds: 5           # Probe every 5s
      timeoutSeconds: 3          # Timeout after 3s
      failureThreshold: 3        # Fail after 3 consecutive failures
      successThreshold: 1        # Success after 1 success
```

### Readiness vs Liveness

```yaml
# Readiness Probe Scenarios:
- Database connection lost (temporary)
- Cache warming up
- External service unavailable
- High load causing slow responses
- Graceful shutdown in progress

# Key Differences:
Liveness:
  - Restarts container on failure
  - successThreshold always 1
  - Used for permanent failures

Readiness:
  - Removes from endpoints on failure
  - Can recover without restart
  - Used for temporary failures
```

---

## 🔹 Startup Probes

### Purpose and Behavior

**Startup Probes** check if a container has started successfully. They disable liveness and readiness probes until the startup probe succeeds, useful for slow-starting containers.

### Basic Startup Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-example
spec:
  containers:
  - name: slow-app
    image: slow-starting-app:v1.0
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      initialDelaySeconds: 10    # Wait 10s before first probe
      periodSeconds: 10          # Probe every 10s
      timeoutSeconds: 5          # Timeout after 5s
      failureThreshold: 30       # Allow 30 failures (5 minutes total)
      successThreshold: 1        # Success after 1 success
    
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
```

### Startup Probe Benefits

```yaml
# Without Startup Probe:
- Liveness probe might kill slow-starting containers
- Need to set high initialDelaySeconds on liveness
- Less responsive to failures after startup

# With Startup Probe:
- Protects slow-starting containers
- Liveness probe can be more aggressive
- Better failure detection after startup
```

---

## 🔹 Probe Types

### HTTP Probes

```yaml
# HTTP GET Probe
livenessProbe:
  httpGet:
    path: /health              # Health check endpoint
    port: 8080                 # Port number or name
    host: 127.0.0.1           # Host (default: pod IP)
    scheme: HTTP              # HTTP or HTTPS
    httpHeaders:              # Custom headers
    - name: Authorization
      value: Bearer token123
    - name: Accept
      value: application/json
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

### TCP Probes

```yaml
# TCP Socket Probe
readinessProbe:
  tcpSocket:
    port: 5432                # Port to check
    host: 127.0.0.1          # Host (default: pod IP)
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

### Command Probes

```yaml
# Exec Command Probe
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - "ps aux | grep '[m]y-process' || exit 1"
  initialDelaySeconds: 30
  periodSeconds: 15
  timeoutSeconds: 10
  failureThreshold: 3
```

### gRPC Probes (Kubernetes 1.24+)

```yaml
# gRPC Probe
readinessProbe:
  grpc:
    port: 9090
    service: myservice        # Optional service name
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

---

## 🔹 Probe Configuration

### Timing Parameters

```yaml
# Probe Timing Configuration
probe:
  initialDelaySeconds: 30     # Delay before first probe
  periodSeconds: 10           # Interval between probes
  timeoutSeconds: 5           # Timeout for each probe
  failureThreshold: 3         # Failures before action
  successThreshold: 1         # Successes to recover (1 for liveness)
  terminationGracePeriodSeconds: 30  # Grace period for termination
```

### Parameter Guidelines

```yaml
# Timing Recommendations:
initialDelaySeconds:
  - Fast apps: 5-15 seconds
  - Medium apps: 15-30 seconds
  - Slow apps: 30-60 seconds
  - Use startup probe for very slow apps

periodSeconds:
  - Liveness: 10-30 seconds
  - Readiness: 5-10 seconds
  - Startup: 10-30 seconds

timeoutSeconds:
  - HTTP probes: 1-5 seconds
  - TCP probes: 1-3 seconds
  - Exec probes: 5-30 seconds

failureThreshold:
  - Liveness: 3-5 failures
  - Readiness: 3 failures
  - Startup: 10-30 failures
```

---

## 🔹 Practical Examples

### 1. Web Application with All Probes

```yaml
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
        image: web-app:v1.0
        ports:
        - containerPort: 8080
        
        # Startup probe for slow initialization
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 18    # 3 minutes total
          successThreshold: 1
        
        # Liveness probe for health monitoring
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            httpHeaders:
            - name: X-Health-Check
              value: liveness
          initialDelaySeconds: 0  # Startup probe handles delay
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        
        # Readiness probe for traffic routing
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
            httpHeaders:
            - name: X-Health-Check
              value: readiness
          initialDelaySeconds: 0  # Startup probe handles delay
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 1
        
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

### 2. Database with TCP and Command Probes

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-db
spec:
  serviceName: postgres
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
          value: myapp
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        
        # Startup probe for database initialization
        startupProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "pg_isready -U admin -d myapp"
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30    # 5 minutes for DB init
        
        # Liveness probe using pg_isready
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "pg_isready -U admin -d myapp"
          initialDelaySeconds: 0
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        
        # Readiness probe using TCP socket
        readinessProbe:
          tcpSocket:
            port: 5432
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        ports:
        - containerPort: 5432
        
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 3. Microservice with Custom Health Endpoints

```yaml
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
      - name: service
        image: microservice:v2.0
        ports:
        - containerPort: 8080
        
        env:
        - name: DB_HOST
          value: postgres-service
        - name: REDIS_HOST
          value: redis-service
        
        # Startup probe with dependency checks
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 20    # 100 seconds total
        
        # Liveness probe for application health
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness probe with dependency checks
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 1
        
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
```

### 4. Legacy Application with Exec Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: legacy-app
spec:
  containers:
  - name: legacy
    image: legacy-app:v1.0
    
    # Startup probe checking process
    startupProbe:
      exec:
        command:
        - /bin/bash
        - -c
        - "pgrep -f 'legacy-process' > /dev/null"
      initialDelaySeconds: 60
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 30
    
    # Liveness probe checking log file
    livenessProbe:
      exec:
        command:
        - /bin/bash
        - -c
        - |
          if [ -f /app/logs/app.log ]; then
            last_log=$(tail -1 /app/logs/app.log | cut -d' ' -f1-2)
            current_time=$(date '+%Y-%m-%d %H:%M')
            if [[ "$last_log" < "$(date -d '5 minutes ago' '+%Y-%m-%d %H:%M')" ]]; then
              exit 1
            fi
          fi
          exit 0
      initialDelaySeconds: 0
      periodSeconds: 60
      timeoutSeconds: 10
      failureThreshold: 3
    
    # Readiness probe checking file existence
    readinessProbe:
      exec:
        command:
        - /bin/bash
        - -c
        - "test -f /app/ready && pgrep -f 'legacy-process'"
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
```

---

## 🔹 Best Practices

### 1. Probe Design Principles

```yaml
# Health Endpoint Design:
/health/startup:
  - Check critical dependencies
  - Database connectivity
  - Required services availability
  - Configuration validation

/health/live:
  - Application core functionality
  - Memory usage within limits
  - Critical threads running
  - No deadlocks detected

/health/ready:
  - All dependencies available
  - Caches warmed up
  - Ready to serve traffic
  - Load within acceptable limits
```

### 2. Timing Configuration

```yaml
# Conservative Approach (Recommended):
startupProbe:
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 30      # 5 minutes total

livenessProbe:
  initialDelaySeconds: 0    # Startup probe handles this
  periodSeconds: 30         # Not too aggressive
  failureThreshold: 3       # Allow some failures

readinessProbe:
  initialDelaySeconds: 0
  periodSeconds: 10         # More frequent for traffic routing
  failureThreshold: 3
```

### 3. Probe Implementation

```yaml
# HTTP Probe Best Practices:
- Use dedicated health endpoints
- Return appropriate HTTP status codes
- Include dependency checks in readiness
- Keep probes lightweight and fast
- Avoid external dependencies in liveness

# Command Probe Best Practices:
- Use simple, fast commands
- Avoid complex shell scripts
- Set appropriate timeouts
- Handle edge cases gracefully
```

### 4. Security Considerations

```yaml
# Secure Health Endpoints:
livenessProbe:
  httpGet:
    path: /internal/health
    port: 8081              # Internal port
    httpHeaders:
    - name: Authorization
      value: Bearer internal-token

# Network Policies:
- Restrict health endpoint access
- Use internal service mesh
- Implement authentication for sensitive checks
```

---

## 🔹 Troubleshooting

### Common Probe Issues

#### 1. Probe Failures

```bash
# Check pod events
kubectl describe pod <pod-name>

# Check probe configuration
kubectl get pod <pod-name> -o yaml | grep -A 20 "livenessProbe\|readinessProbe\|startupProbe"

# Common causes:
# - Incorrect probe configuration
# - Application not responding on expected port
# - Health endpoint returning wrong status code
# - Timeout too short for application response
```

#### 2. Restart Loops

```bash
# Check restart count
kubectl get pods

# Check logs from previous container
kubectl logs <pod-name> --previous

# Common causes:
# - Liveness probe too aggressive
# - Application startup time longer than probe allows
# - Health endpoint not implemented correctly
```

#### 3. Traffic Not Reaching Pods

```bash
# Check service endpoints
kubectl get endpoints <service-name>

# Check readiness probe status
kubectl describe pod <pod-name> | grep -A 5 "Readiness"

# Common causes:
# - Readiness probe failing
# - Health endpoint not accessible
# - Dependencies not available
```

### Debugging Commands

```bash
# Pod status and events
kubectl get pods -o wide
kubectl describe pod <pod-name>

# Check probe configuration
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].livenessProbe}'

# Test health endpoints manually
kubectl port-forward pod/<pod-name> 8080:8080
curl http://localhost:8080/health

# Check service endpoints
kubectl get endpoints
kubectl describe service <service-name>

# Monitor probe failures
kubectl get events --field-selector involvedObject.name=<pod-name>
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: Explain the difference between liveness, readiness, and startup probes.**
- **Liveness**: Restarts container if unhealthy (permanent failures)
- **Readiness**: Removes from service if not ready (temporary failures)
- **Startup**: Protects slow-starting containers from premature liveness checks

**Q2: When would you use each type of probe?**
- **Liveness**: Deadlocks, memory leaks, hung processes
- **Readiness**: Dependency failures, cache warming, high load
- **Startup**: Slow application initialization, database migrations

**Q3: What happens when probes fail?**
- **Liveness failure**: Container restart
- **Readiness failure**: Removed from service endpoints
- **Startup failure**: Container restart (after failureThreshold)

### Technical Questions

**Q4: How do you design effective health check endpoints?**
```yaml
# Liveness endpoint (/health/live):
- Check core application functionality
- Avoid external dependencies
- Return quickly (< 1 second)
- Use HTTP 200 for healthy, 5xx for unhealthy

# Readiness endpoint (/health/ready):
- Check all dependencies
- Verify cache status
- Check database connectivity
- Return 200 when ready to serve traffic
```

**Q5: What are the probe timing best practices?**
- Use startup probes for slow-starting applications
- Set conservative timeouts initially
- Monitor and adjust based on application behavior
- Avoid too aggressive liveness probes

**Q6: How do you troubleshoot probe failures?**
```bash
# Check pod events and logs
# Verify probe configuration
# Test endpoints manually
# Check application startup time
# Monitor resource usage
```

### Scenario-Based Questions

**Q7: Design probes for a microservice with database dependencies.**
```yaml
# Use startup probe for initial DB connection
# Liveness probe for application health only
# Readiness probe includes DB connectivity check
# Separate internal health endpoints
```

**Q8: How would you handle a legacy application without health endpoints?**
```yaml
# Use exec probes to check processes
# Monitor log files for activity
# Check port availability with TCP probes
# Implement wrapper scripts for health checks
```

**Q9: Troubleshoot pods stuck in restart loops.**
```bash
# Check liveness probe configuration
# Verify application startup time
# Check resource limits and requests
# Review application logs
# Consider using startup probe
```

### Best Practices Questions

**Q10: What are probe configuration best practices?**
- Always implement health endpoints
- Use appropriate probe types for each scenario
- Set conservative timing initially
- Monitor probe success rates
- Implement proper logging and metrics
- Test probe behavior under load
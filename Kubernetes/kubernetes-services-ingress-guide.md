# 🌐 Kubernetes Services & Ingress - DevOps Interview Guide

## Table of Contents
1. [Services Overview](#services-overview)
2. [Service Types](#service-types)
3. [Service Discovery](#service-discovery)
4. [Ingress Overview](#ingress-overview)
5. [Ingress Controllers](#ingress-controllers)
6. [Advanced Ingress Features](#advanced-ingress-features)
7. [Practical Examples](#practical-examples)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)
10. [Interview Questions](#interview-questions)

---

## 🔹 Services Overview

### What are Kubernetes Services?

**Services** provide stable network endpoints for accessing pods, abstracting away the ephemeral nature of pod IPs and enabling load balancing.

### Service Fundamentals

```yaml
# Service Problems Solved:
Pod Ephemeral Nature:
  - Pod IPs change on restart
  - Pods can be scheduled on different nodes
  - No stable endpoint for communication

Load Balancing:
  - Distribute traffic across multiple pods
  - Health check integration
  - Session affinity options

Service Discovery:
  - DNS-based service resolution
  - Environment variable injection
  - Cluster-internal communication
```

### Service Components

```yaml
# Service Architecture:
Selector: Matches pods using labels
Endpoints: List of pod IPs that match selector
ClusterIP: Virtual IP for internal access
Port Mapping: Service port to target port mapping
Load Balancer: Distributes traffic to healthy pods
```

---

## 🔹 Service Types

### ClusterIP (Default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
  labels:
    app: web-application
    component: frontend
spec:
  type: ClusterIP
  selector:
    app: web-app
    tier: frontend
  ports:
  - name: http
    port: 80          # Service port
    targetPort: 8080  # Pod port
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  
  # Optional: Session affinity
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
```

### NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30080    # Optional: specify port (30000-32767)
    protocol: TCP
  
  # External traffic policy
  externalTrafficPolicy: Local  # Preserve source IP
```

### LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  annotations:
    # Cloud provider specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:region:account:certificate/cert-id"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8080
  
  # Load balancer source ranges
  loadBalancerSourceRanges:
  - 10.0.0.0/8
  - 192.168.0.0/16
  
  # External traffic policy
  externalTrafficPolicy: Local
```

### ExternalName

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.example.com
  ports:
  - port: 5432
    targetPort: 5432
---
# Usage in pods
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:v1.0
    env:
    - name: DB_HOST
      value: "external-database.default.svc.cluster.local"
```

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None    # Headless service
  selector:
    app: stateful-app
  ports:
  - port: 80
    targetPort: 8080
---
# Returns individual pod IPs instead of service IP
# Used with StatefulSets for direct pod access
# DNS returns A records for each pod
```

---

## 🔹 Service Discovery

### DNS-Based Discovery

```yaml
# Service DNS Names:
# <service-name>.<namespace>.svc.cluster.local

# Same namespace (short form):
http://web-service:80

# Different namespace (full form):
http://web-service.production.svc.cluster.local:80

# Headless service (returns pod IPs):
web-0.headless-service.default.svc.cluster.local
web-1.headless-service.default.svc.cluster.local
```

### Environment Variables

```yaml
# Kubernetes automatically creates environment variables
# for services that exist when pod is created

# For service named "web-service" on port 80:
WEB_SERVICE_SERVICE_HOST=10.96.0.1
WEB_SERVICE_SERVICE_PORT=80
WEB_SERVICE_PORT=tcp://10.96.0.1:80
WEB_SERVICE_PORT_80_TCP=tcp://10.96.0.1:80
WEB_SERVICE_PORT_80_TCP_PROTO=tcp
WEB_SERVICE_PORT_80_TCP_PORT=80
WEB_SERVICE_PORT_80_TCP_ADDR=10.96.0.1
```

### Service Endpoints

```yaml
# Endpoints are automatically managed
apiVersion: v1
kind: Endpoints
metadata:
  name: web-service
subsets:
- addresses:
  - ip: 10.244.1.5
    targetRef:
      kind: Pod
      name: web-app-1
  - ip: 10.244.2.3
    targetRef:
      kind: Pod
      name: web-app-2
  ports:
  - name: http
    port: 8080
    protocol: TCP
```

---

## 🔹 Ingress Overview

### What is Ingress?

**Ingress** manages external access to services, typically HTTP/HTTPS, providing load balancing, SSL termination, and name-based virtual hosting.

### Ingress vs Service

| Aspect | Service | Ingress |
|--------|---------|---------|
| **Layer** | L4 (Transport) | L7 (Application) |
| **Protocols** | TCP/UDP | HTTP/HTTPS |
| **Features** | Load balancing | SSL, routing, redirects |
| **External Access** | LoadBalancer/NodePort | HTTP(S) routing |
| **Cost** | Per service LB | Single LB for multiple services |

### Basic Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  # TLS configuration
  tls:
  - hosts:
    - api.example.com
    - www.example.com
    secretName: tls-secret
  
  # Ingress rules
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## 🔹 Ingress Controllers

### NGINX Ingress Controller

```yaml
# NGINX Ingress with advanced features
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    # Basic annotations
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/rate-limit-connections: "10"
    nginx.ingress.kubernetes.io/rate-limit-requests-per-minute: "60"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization"
    
    # Custom configuration
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /api/v2(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
```

### Traefik Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.middlewares: default-auth@kubernetescrd
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### AWS Load Balancer Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aws-ingress
  annotations:
    # ALB annotations
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/cert-id
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    
    # Health check
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '30'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '3'
    
    # Tags
    alb.ingress.kubernetes.io/tags: Environment=production,Team=platform
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## 🔹 Advanced Ingress Features

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      # Exact path matching
      - path: /api/v1/users
        pathType: Exact
        backend:
          service:
            name: user-service
            port:
              number: 80
      
      # Prefix path matching
      - path: /api/v1/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      
      # Implementation-specific matching
      - path: /api/v1/products.*
        pathType: ImplementationSpecific
        backend:
          service:
            name: product-service
            port:
              number: 80
```

### SSL/TLS Configuration

```yaml
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # Base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi... # Base64 encoded private key
---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512"
spec:
  tls:
  - hosts:
    - secure.example.com
    secretName: tls-secret
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 80
```

### Authentication and Authorization

```yaml
# Basic Auth Secret
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: Opaque
data:
  auth: YWRtaW46JGFwcjEkSDY1... # htpasswd generated hash
---
# Ingress with authentication
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required - Admin Area"
spec:
  rules:
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

### Canary Deployments

```yaml
# Production Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service-v1
            port:
              number: 80
---
# Canary Ingress (10% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
    # Alternative canary rules:
    # nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    # nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service-v2
            port:
              number: 80
```

---

## 🔹 Practical Examples

### 1. Complete Web Application Stack

```yaml
# Backend Service
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
---
# Frontend Service
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
---
# API Service
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 8000
    targetPort: 8000
---
# Main Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    - api.myapp.example.com
    secretName: myapp-tls
  rules:
  # Frontend
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  # API
  - host: api.myapp.example.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8000
      - path: /backend
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8080
```

### 2. Multi-Environment Setup

```yaml
# Development Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-ingress
  namespace: development
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: dev-auth
spec:
  ingressClassName: nginx
  rules:
  - host: dev.myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
---
# Staging Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: staging-ingress
  namespace: staging
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - staging.myapp.example.com
    secretName: staging-tls
  rules:
  - host: staging.myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
---
# Production Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rate-limit-connections: "20"
    nginx.ingress.kubernetes.io/rate-limit-requests-per-minute: "300"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: prod-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### 3. Microservices with Service Mesh

```yaml
# User Service
apiVersion: v1
kind: Service
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 8080
    targetPort: 8080
---
# Order Service
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 8080
    targetPort: 8080
---
# Payment Service
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  labels:
    app: payment-service
spec:
  selector:
    app: payment-service
  ports:
  - port: 8080
    targetPort: 8080
---
# API Gateway Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.microservices.example.com
    secretName: microservices-tls
  rules:
  - host: api.microservices.example.com
    http:
      paths:
      - path: /users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
      - path: /payments(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 8080
```

### 4. Blue-Green Deployment

```yaml
# Blue Service (Current Production)
apiVersion: v1
kind: Service
metadata:
  name: app-service-blue
spec:
  selector:
    app: myapp
    version: blue
  ports:
  - port: 80
    targetPort: 8080
---
# Green Service (New Version)
apiVersion: v1
kind: Service
metadata:
  name: app-service-green
spec:
  selector:
    app: myapp
    version: green
  ports:
  - port: 80
    targetPort: 8080
---
# Production Ingress (Blue)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service-blue  # Switch to green for deployment
            port:
              number: 80
---
# Testing Ingress (Green)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: testing-ingress
spec:
  rules:
  - host: test.myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service-green
            port:
              number: 80
```

---

## 🔹 Best Practices

### 1. Service Design

```yaml
# Service Best Practices:
- Use meaningful service names
- Set appropriate session affinity
- Configure health checks properly
- Use headless services for StatefulSets
- Implement proper port naming

# Port Naming Convention:
ports:
- name: http      # Standard HTTP
- name: https     # Standard HTTPS
- name: grpc      # gRPC protocol
- name: metrics   # Prometheus metrics
- name: admin     # Admin interface
```

### 2. Ingress Configuration

```yaml
# Ingress Best Practices:
- Use TLS for all external traffic
- Implement rate limiting
- Configure proper timeouts
- Use authentication where needed
- Monitor ingress metrics

# Security Headers:
nginx.ingress.kubernetes.io/configuration-snippet: |
  more_set_headers "X-Frame-Options: DENY";
  more_set_headers "X-Content-Type-Options: nosniff";
  more_set_headers "X-XSS-Protection: 1; mode=block";
  more_set_headers "Strict-Transport-Security: max-age=31536000";
```

### 3. SSL/TLS Management

```yaml
# Use cert-manager for automatic certificates
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

### 4. Monitoring and Observability

```yaml
# Service monitoring annotations
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"

# Ingress monitoring
nginx.ingress.kubernetes.io/enable-access-log: "true"
nginx.ingress.kubernetes.io/enable-rewrite-log: "true"
```

---

## 🔹 Troubleshooting

### Common Service Issues

#### 1. Service Not Accessible

```bash
# Check service endpoints
kubectl get endpoints my-service

# Check service configuration
kubectl describe service my-service

# Test service connectivity
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://my-service:80

# Common causes:
# - Selector doesn't match pod labels
# - Pods not ready/healthy
# - Port configuration mismatch
# - Network policies blocking traffic
```

#### 2. Ingress Not Working

```bash
# Check ingress status
kubectl describe ingress my-ingress

# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Test DNS resolution
nslookup myapp.example.com

# Check TLS certificate
openssl s_client -connect myapp.example.com:443 -servername myapp.example.com
```

#### 3. Load Balancer Issues

```bash
# Check load balancer status
kubectl get service my-lb-service

# Check cloud provider events
kubectl describe service my-lb-service

# Check security groups/firewall rules
# Verify health check configuration
```

### Debugging Commands

```bash
# Service debugging
kubectl get services -A
kubectl describe service my-service
kubectl get endpoints my-service

# Ingress debugging
kubectl get ingress -A
kubectl describe ingress my-ingress
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Network connectivity
kubectl run debug-pod --image=busybox -it --rm -- sh
# Inside pod: wget, nslookup, telnet commands

# DNS debugging
kubectl run dns-test --image=busybox -it --rm -- nslookup kubernetes.default
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: Explain the difference between Services and Ingress.**
- **Services**: L4 load balancing, internal/external access, TCP/UDP
- **Ingress**: L7 HTTP(S) routing, SSL termination, path-based routing

**Q2: What are the different Service types and when to use each?**
- **ClusterIP**: Internal cluster communication
- **NodePort**: External access via node ports
- **LoadBalancer**: Cloud provider load balancer
- **ExternalName**: DNS alias for external services

**Q3: How does service discovery work in Kubernetes?**
- DNS-based: service-name.namespace.svc.cluster.local
- Environment variables for existing services
- Endpoints automatically managed by controller

### Technical Questions

**Q4: How do you configure SSL termination in Ingress?**
```yaml
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
```

**Q5: What is session affinity and how do you configure it?**
```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

**Q6: How do you implement canary deployments with Ingress?**
```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "10"
```

### Scenario-Based Questions

**Q7: Design ingress for a multi-tenant application.**
```yaml
# Host-based routing per tenant
# Path-based routing for services
# Authentication per tenant
# Rate limiting per tenant
# SSL certificates per domain
```

**Q8: How would you troubleshoot intermittent service connectivity?**
```bash
# Check service endpoints stability
# Monitor pod health and readiness
# Check network policies
# Verify DNS resolution
# Test load balancer health checks
```

**Q9: Implement blue-green deployment strategy.**
```yaml
# Two identical environments (blue/green)
# Switch ingress backend service
# Test green environment before switch
# Quick rollback capability
# Zero-downtime deployment
```

### Best Practices Questions

**Q10: What are service and ingress security best practices?**
- Use TLS for all external traffic
- Implement authentication and authorization
- Configure rate limiting and DDoS protection
- Use network policies for traffic control
- Regular security scanning and updates
- Monitor and log all access attempts
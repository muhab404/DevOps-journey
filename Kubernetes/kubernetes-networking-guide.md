# 🌐 Kubernetes Networking - DevOps Interview Guide

## Table of Contents
1. [Networking Overview](#networking-overview)
2. [Pod Networking](#pod-networking)
3. [Services](#services)
4. [Ingress](#ingress)
5. [Network Policies](#network-policies)
6. [DNS](#dns)
7. [Load Balancing](#load-balancing)
8. [Service Mesh](#service-mesh)
9. [CNI Plugins](#cni-plugins)
10. [Troubleshooting](#troubleshooting)
11. [Interview Questions](#interview-questions)

---

## 🔹 Networking Overview

### Kubernetes Network Model

```yaml
# Network Requirements:
Pod-to-Pod Communication:
  - Every pod gets unique IP address
  - Pods can communicate without NAT
  - Cross-node communication works seamlessly

Service Discovery:
  - Services provide stable endpoints
  - DNS-based service discovery
  - Load balancing across pod replicas

External Access:
  - Ingress controllers for HTTP/HTTPS
  - LoadBalancer services for TCP/UDP
  - NodePort for direct node access
```

### Network Architecture

```yaml
# Network Components:
Container Network Interface (CNI):
  - Flannel, Calico, Weave, Cilium
  - Handles pod IP allocation
  - Manages network routing

kube-proxy:
  - Service load balancing
  - iptables/IPVS rules
  - Traffic forwarding

CoreDNS:
  - Service discovery
  - DNS resolution
  - Custom DNS policies

Ingress Controllers:
  - NGINX, Traefik, HAProxy
  - SSL termination
  - Path-based routing
```

---

## 🔹 Pod Networking

### Pod Network Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-pod
  labels:
    app: web-server
spec:
  # Host networking (use node's network)
  hostNetwork: false
  
  # DNS configuration
  dnsPolicy: ClusterFirst
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - my.dns.search.suffix
    options:
    - name: ndots
      value: "2"
    - name: edns0
  
  containers:
  - name: web
    image: nginx:1.21
    ports:
    - name: http
      containerPort: 80
      protocol: TCP
    - name: https
      containerPort: 443
      protocol: TCP
    
    # Environment variables for networking
    env:
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
```

### Multi-Container Pod Networking

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-network
spec:
  containers:
  # Main application container
  - name: app
    image: nginx:1.21
    ports:
    - containerPort: 80
    
  # Sidecar proxy container
  - name: proxy
    image: envoyproxy/envoy:v1.24.0
    ports:
    - containerPort: 8080
    - containerPort: 9901  # Admin interface
    
    # Proxy configuration
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
    
    command: ["/usr/local/bin/envoy"]
    args:
    - "--config-path"
    - "/etc/envoy/envoy.yaml"
    - "--service-cluster"
    - "proxy-cluster"
  
  volumes:
  - name: envoy-config
    configMap:
      name: envoy-config
```

### Host Network Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-network-pod
spec:
  # Use host network namespace
  hostNetwork: true
  hostPID: false
  
  # DNS policy for host network
  dnsPolicy: ClusterFirstWithHostNet
  
  containers:
  - name: network-tool
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
    
    # Security context for network access
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "NET_RAW"]
```

---

## 🔹 Services

### ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  labels:
    app: backend
spec:
  # Default service type
  type: ClusterIP
  
  # Service IP (optional, auto-assigned if not specified)
  clusterIP: 10.96.100.100
  
  # Pod selector
  selector:
    app: backend
    tier: api
  
  ports:
  - name: http
    port: 80          # Service port
    targetPort: 8080  # Container port
    protocol: TCP
  - name: grpc
    port: 9090
    targetPort: grpc  # Named port reference
    protocol: TCP
  
  # Session affinity
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

### NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  
  selector:
    app: frontend
  
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30080    # External port on nodes (30000-32767)
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    nodePort: 30443
    protocol: TCP
  
  # External traffic policy
  externalTrafficPolicy: Local  # Preserve source IP
```

### LoadBalancer Service

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
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
spec:
  type: LoadBalancer
  
  # Load balancer IP (cloud provider specific)
  loadBalancerIP: 203.0.113.100
  
  # Source IP ranges allowed
  loadBalancerSourceRanges:
  - 10.0.0.0/8
  - 192.168.0.0/16
  
  selector:
    app: web-app
  
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  
  externalTrafficPolicy: Local
```

### ExternalName Service

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
```

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  # Headless service (no cluster IP)
  clusterIP: None
  
  selector:
    app: database
    statefulset: postgres
  
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
  
  # Publish not ready addresses
  publishNotReadyAddresses: true
```

### Service with Multiple Selectors

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-version-service
spec:
  selector:
    app: web-app
    # No version selector - routes to all versions
  
  ports:
  - port: 80
    targetPort: 8080

---
# Canary service for specific version
apiVersion: v1
kind: Service
metadata:
  name: canary-service
spec:
  selector:
    app: web-app
    version: canary
  
  ports:
  - port: 80
    targetPort: 8080
```

---

## 🔹 Ingress

### Basic Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  # Ingress class
  ingressClassName: nginx
  
  # TLS configuration
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: example-tls
  
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
```

### Advanced Ingress with Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  annotations:
    # NGINX specific annotations
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    
    # Load balancing
    nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"
    nginx.ingress.kubernetes.io/load-balance: "round_robin"
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
    
    # Canary deployment
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      
      - path: /static(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: static-service
            port:
              number: 80
```

### Ingress with Multiple Backends

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  
  tls:
  - hosts:
    - microservices.example.com
    secretName: microservices-tls
  
  rules:
  - host: microservices.example.com
    http:
      paths:
      # User service
      - path: /users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      
      # Order service
      - path: /orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
      
      # Payment service
      - path: /payments(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 8080
      
      # Default backend
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

## 🔹 Network Policies

### Default Deny Network Policy

```yaml
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
```

### Allow Ingress from Specific Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
      tier: api
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    # Allow from frontend pods in same namespace
    - podSelector:
        matchLabels:
          app: frontend
    
    # Allow from ingress controller namespace
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    
    # Allow from monitoring namespace
    - namespaceSelector:
        matchLabels:
          name: monitoring
    
    ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 9090  # Metrics port
```

### Allow Egress to Specific Destinations

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  
  policyTypes:
  - Egress
  
  egress:
  # Allow DNS resolution
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  
  # Allow database access
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow Redis access
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  
  # Allow external HTTPS API calls
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 192.168.0.0/16
        - 172.16.0.0/12
    ports:
    - protocol: TCP
      port: 443
```

### Complex Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-tier-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: web
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Allow from load balancer
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  
  # Allow health checks from kubelet
  - from: []
    ports:
    - protocol: TCP
      port: 8080  # Health check port
  
  egress:
  # DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53
  
  # Backend API
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8080
  
  # External CDN
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
```

---

## 🔹 DNS

### CoreDNS Configuration

```yaml
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
    
    # Custom domain
    example.com:53 {
        errors
        cache 30
        forward . 8.8.8.8 8.8.4.4
    }
```

### Custom DNS Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  # DNS policy options:
  # - Default: Use node's DNS
  # - ClusterFirst: Use cluster DNS first
  # - ClusterFirstWithHostNet: For hostNetwork pods
  # - None: Use dnsConfig only
  dnsPolicy: None
  
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    - 1.1.1.1
    
    searches:
    - my.dns.search.suffix
    - another.search.domain
    
    options:
    - name: ndots
      value: "2"
    - name: edns0
    - name: timeout
      value: "5"
    - name: attempts
      value: "3"
  
  containers:
  - name: app
    image: nginx:1.21
```

### Service DNS Records

```yaml
# Service creates DNS records:
# <service-name>.<namespace>.svc.cluster.local

# Examples:
# frontend.production.svc.cluster.local
# backend.production.svc.cluster.local
# database.production.svc.cluster.local

# Headless service creates records for each pod:
# <pod-name>.<service-name>.<namespace>.svc.cluster.local

# StatefulSet pod DNS:
# <pod-name>.<service-name>.<namespace>.svc.cluster.local
# web-0.nginx.default.svc.cluster.local
# web-1.nginx.default.svc.cluster.local
```

---

## 🔹 Load Balancing

### Service Load Balancing

```yaml
apiVersion: v1
kind: Service
metadata:
  name: load-balanced-service
spec:
  selector:
    app: web-app
  
  ports:
  - port: 80
    targetPort: 8080
  
  # Session affinity options
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  
  # External traffic policy
  externalTrafficPolicy: Local  # Preserves source IP
```

### Ingress Load Balancing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: load-balanced-ingress
  annotations:
    # Load balancing algorithms
    nginx.ingress.kubernetes.io/load-balance: "round_robin"  # round_robin, least_conn, ip_hash
    nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"
    
    # Sticky sessions
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/affinity-mode: "persistent"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-expires: "86400"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "86400"
    nginx.ingress.kubernetes.io/session-cookie-path: "/"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
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

## 🔹 Service Mesh

### Istio Service Mesh

```yaml
# Enable Istio injection
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
```

```yaml
# Virtual Service for traffic routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web-app-vs
  namespace: production
spec:
  hosts:
  - web-app
  - web-app.production.svc.cluster.local
  
  http:
  # Canary routing
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: web-app
        subset: canary
      weight: 100
  
  # Default routing with traffic split
  - route:
    - destination:
        host: web-app
        subset: stable
      weight: 90
    - destination:
        host: web-app
        subset: canary
      weight: 10
    
    # Fault injection for testing
    fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
      abort:
        percentage:
          value: 0.1
        httpStatus: 500
```

```yaml
# Destination Rule for load balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: web-app-dr
  namespace: production
spec:
  host: web-app
  
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN  # ROUND_ROBIN, LEAST_CONN, RANDOM, PASSTHROUGH
    
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
```

```yaml
# Gateway for external traffic
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-app-gateway
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
    - app.example.com
    tls:
      httpsRedirect: true
  
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - app.example.com
    tls:
      mode: SIMPLE
      credentialName: app-tls-secret
```

---

## 🔹 CNI Plugins

### Calico Network Policy

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: advanced-calico-policy
  namespace: production
spec:
  selector: app == 'web-app'
  
  types:
  - Ingress
  - Egress
  
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == 'frontend'
    destination:
      ports:
      - 8080
  
  - action: Allow
    protocol: TCP
    source:
      namespaceSelector: name == 'monitoring'
    destination:
      ports:
      - 9090
  
  egress:
  - action: Allow
    protocol: UDP
    destination:
      ports:
      - 53
  
  - action: Allow
    protocol: TCP
    destination:
      selector: app == 'database'
      ports:
      - 5432
  
  - action: Deny
    protocol: TCP
    destination:
      nets:
      - 192.168.1.0/24
```

### Cilium Network Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: cilium-l7-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/.*"
        - method: "POST"
          path: "/api/v1/users"
          headers:
          - "Content-Type: application/json"
  
  egress:
  - toEndpoints:
    - matchLabels:
        app: database
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
  
  - toFQDNs:
    - matchName: "api.external.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

---

## 🔹 Troubleshooting

### Network Debugging Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
    
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "NET_RAW"]
    
    # Useful environment variables
    env:
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
```

### Common Network Debugging Commands

```bash
# Inside network debug pod:

# Check pod connectivity
ping <pod-ip>
ping <service-name>.<namespace>.svc.cluster.local

# DNS resolution
nslookup <service-name>
dig <service-name>.<namespace>.svc.cluster.local

# Port connectivity
telnet <service-ip> <port>
nc -zv <service-ip> <port>

# Network routes
ip route show
netstat -rn

# Network interfaces
ip addr show
ifconfig

# iptables rules (if accessible)
iptables -L -n -v

# Check service endpoints
kubectl get endpoints <service-name>

# Check network policies
kubectl get networkpolicies
kubectl describe networkpolicy <policy-name>
```

---

## 🔹 Interview Questions

### Basic Networking Questions

**Q: Explain the Kubernetes network model.**
A: Kubernetes follows a flat network model where:
- Every pod gets a unique IP address
- Pods can communicate without NAT
- Services provide stable endpoints
- No port conflicts between pods

**Q: What are the different service types?**
A: Four service types:
- **ClusterIP**: Internal cluster access only
- **NodePort**: Exposes service on each node's IP
- **LoadBalancer**: Cloud provider load balancer
- **ExternalName**: Maps to external DNS name

**Q: How does service discovery work?**
A: Through DNS and environment variables:
- DNS: `<service>.<namespace>.svc.cluster.local`
- Environment variables: `<SERVICE>_SERVICE_HOST/PORT`
- CoreDNS provides cluster DNS resolution

### Advanced Networking Questions

**Q: Explain how kube-proxy works.**
A: kube-proxy implements service load balancing:
- **iptables mode**: Creates iptables rules for traffic forwarding
- **IPVS mode**: Uses IPVS for better performance and load balancing algorithms
- **userspace mode**: Legacy mode, proxies traffic in userspace

**Q: What are Network Policies and how do they work?**
A: Network Policies provide network segmentation:
- Define ingress/egress rules for pods
- Use label selectors to target pods
- Require CNI plugin support (Calico, Cilium, etc.)
- Default allow-all behavior without policies

**Q: How does Ingress differ from Services?**
A: Key differences:
- **Services**: L4 load balancing, cluster-internal
- **Ingress**: L7 load balancing, HTTP/HTTPS routing
- **Ingress**: Path-based routing, SSL termination
- **Services**: TCP/UDP support, simpler configuration

### Troubleshooting Questions

**Q: Pod cannot reach external services**
A: Check:
1. Network policies blocking egress
2. DNS resolution issues
3. Firewall rules on nodes
4. CNI plugin configuration
5. Service mesh policies

**Q: Service not accessible from pods**
A: Debug steps:
1. Check service endpoints: `kubectl get endpoints`
2. Verify pod labels match service selector
3. Test DNS resolution: `nslookup service-name`
4. Check network policies
5. Verify kube-proxy is running

**Q: Ingress returns 503 errors**
A: Common causes:
1. Backend service not ready
2. Incorrect service/port configuration
3. Ingress controller issues
4. SSL certificate problems
5. Network policies blocking traffic

This comprehensive networking guide covers all essential Kubernetes networking concepts for senior DevOps engineer interviews, including practical examples, troubleshooting scenarios, and real-world configurations.
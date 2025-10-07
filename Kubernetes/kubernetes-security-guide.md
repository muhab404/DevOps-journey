# 🔐 Kubernetes Security - DevOps Interview Guide

## Table of Contents
1. [Security Overview](#security-overview)
2. [Authentication & Authorization](#authentication--authorization)
3. [Pod Security](#pod-security)
4. [Network Security](#network-security)
5. [Secrets Management](#secrets-management)
6. [Image Security](#image-security)
7. [Admission Controllers](#admission-controllers)
8. [Security Contexts](#security-contexts)
9. [Service Mesh Security](#service-mesh-security)
10. [Security Monitoring](#security-monitoring)
11. [Best Practices](#best-practices)
12. [Interview Questions](#interview-questions)

---

## 🔹 Security Overview

### Kubernetes Security Layers

```yaml
# 4C's of Cloud Native Security:
Cloud: Infrastructure security (AWS, GCP, Azure)
Cluster: Kubernetes cluster security
Container: Container runtime security
Code: Application code security

# Defense in Depth:
Network Level: Network policies, service mesh
Cluster Level: RBAC, admission controllers
Node Level: Node security, runtime protection
Pod Level: Security contexts, pod security standards
Application Level: Secrets, TLS, authentication
```

### Security Architecture

```yaml
# Security Components:
API Server Security:
  - Authentication (who you are)
  - Authorization (what you can do)
  - Admission Control (policy enforcement)

Pod Security:
  - Security contexts
  - Pod security standards
  - Runtime security

Network Security:
  - Network policies
  - Service mesh (mTLS)
  - Ingress security

Data Security:
  - Secrets management
  - Encryption at rest
  - Encryption in transit
```

---

## 🔹 Authentication & Authorization

### RBAC (Role-Based Access Control)

```yaml
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
automountServiceAccountToken: true
secrets:
- name: app-service-account-token
```

```yaml
# Role - Namespace scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-manager
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/status"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
```

```yaml
# ClusterRole - Cluster scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-monitor
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/status", "nodes/metrics"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
```

```yaml
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-manager-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
- kind: User
  name: devops-user
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: devops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-monitor-binding
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: cluster-monitor
  apiGroup: rbac.authorization.k8s.io
```

### Authentication Methods

```yaml
# X.509 Client Certificates
apiVersion: v1
kind: Config
users:
- name: devops-user
  user:
    client-certificate: /path/to/client.crt
    client-key: /path/to/client.key

# Service Account Tokens
apiVersion: v1
kind: Config
users:
- name: service-account-user
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9...

# OIDC Integration
apiVersion: v1
kind: Config
users:
- name: oidc-user
  user:
    auth-provider:
      name: oidc
      config:
        client-id: kubernetes
        client-secret: client-secret
        id-token: eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9...
        idp-issuer-url: https://accounts.google.com
        refresh-token: refresh-token
```

---

## 🔹 Pod Security

### Security Contexts

```yaml
# Pod Security Context
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod-level security context
  securityContext:
    # Run as non-root user
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    
    # Filesystem permissions
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"
    
    # Supplemental groups
    supplementalGroups: [4000, 5000]
    
    # SELinux options
    seLinuxOptions:
      level: "s0:c123,c456"
    
    # Seccomp profile
    seccompProfile:
      type: RuntimeDefault
    
    # Sysctls
    sysctls:
    - name: net.core.somaxconn
      value: "1024"
  
  containers:
  - name: app
    image: nginx:1.21
    
    # Container-level security context
    securityContext:
      # Capabilities
      capabilities:
        add: ["NET_ADMIN"]
        drop: ["ALL"]
      
      # Privilege escalation
      allowPrivilegeEscalation: false
      
      # Privileged mode
      privileged: false
      
      # Read-only root filesystem
      readOnlyRootFilesystem: true
      
      # Run as specific user
      runAsNonRoot: true
      runAsUser: 1001
      runAsGroup: 3001
      
      # Seccomp profile
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/audit.json
    
    # Volume mounts for writable directories
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
  
  volumes:
  - name: tmp-volume
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
  - name: var-run
    emptyDir: {}
```

### Pod Security Standards

```yaml
# Pod Security Standards Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    # Enforce restricted security standard
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    
    # Version
    pod-security.kubernetes.io/enforce-version: v1.25
```

```yaml
# Restricted Pod Example
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: secure-namespace
spec:
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
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop: ["ALL"]
      seccompProfile:
        type: RuntimeDefault
    
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
    
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

---

## 🔹 Network Security

### Network Policies

```yaml
# Default Deny All Network Policy
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

```yaml
# Allow Specific Ingress Traffic
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
    # Allow from frontend pods
    - podSelector:
        matchLabels:
          app: frontend
          tier: web
    # Allow from specific namespace
    - namespaceSelector:
        matchLabels:
          name: frontend-namespace
    # Allow from specific IP ranges
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.1.0/24
    
    ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 8443
```

```yaml
# Allow Specific Egress Traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-egress
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
          app: database
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow external API calls
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
    - protocol: TCP
      port: 80
```

```yaml
# Complex Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Allow ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  
  # Allow monitoring
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    - podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090
  
  egress:
  # DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53
  
  # Database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  
  # Redis cache
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
```

---

## 🔹 Secrets Management

### Kubernetes Secrets

```yaml
# Generic Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  # Base64 encoded values
  database-url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0BkYi5leGFtcGxlLmNvbS9kYg==
  api-key: YWJjZGVmZ2hpams=
  private-key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0t...
stringData:
  # Plain text values (automatically base64 encoded)
  config.yaml: |
    database:
      host: db.example.com
      port: 5432
      name: myapp
    redis:
      host: redis.example.com
      port: 6379
```

```yaml
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0t...
```

```yaml
# Docker Registry Secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5leGFtcGxlLmNvbSI6eyJ1c2VybmFtZSI6InVzZXIiLCJwYXNzd29yZCI6InBhc3MiLCJhdXRoIjoiZFhObGNqcHdZWE56In19fQ==
```

### Using Secrets in Pods

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
    image: registry.example.com/myapp:v1.0
    
    # Environment variables from secrets
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-url
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
    
    # All secret keys as environment variables
    envFrom:
    - secretRef:
        name: app-secrets
        optional: false
    
    # Mount secrets as files
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
    - name: tls-volume
      mountPath: /etc/tls
      readOnly: true
  
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secrets
      defaultMode: 0400
      items:
      - key: private-key
        path: private.key
        mode: 0400
      - key: config.yaml
        path: config.yaml
        mode: 0444
  
  - name: tls-volume
    secret:
      secretName: tls-secret
      defaultMode: 0400
```

### External Secrets Operator

```yaml
# SecretStore for AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-store
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
      auth:
        secretRef:
          accessKeyID:
            name: aws-credentials
            key: access-key-id
          secretAccessKey:
            name: aws-credentials
            key: secret-access-key
```

```yaml
# External Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-external-secret
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  
  target:
    name: app-secrets
    creationPolicy: Owner
  
  data:
  - secretKey: database-password
    remoteRef:
      key: prod/database
      property: password
  - secretKey: api-key
    remoteRef:
      key: prod/api
      property: key
```

---

## 🔹 Image Security

### Image Security Policies

```yaml
# Pod with secure image practices
apiVersion: v1
kind: Pod
metadata:
  name: secure-image-pod
spec:
  containers:
  - name: app
    # Use specific image tags, not 'latest'
    image: nginx:1.21.6-alpine
    
    # Image pull policy
    imagePullPolicy: Always
    
    securityContext:
      # Run as non-root
      runAsNonRoot: true
      runAsUser: 1000
      
      # Read-only filesystem
      readOnlyRootFilesystem: true
      
      # Drop all capabilities
      capabilities:
        drop: ["ALL"]
  
  # Image pull secrets
  imagePullSecrets:
  - name: private-registry-secret
```

### OPA Gatekeeper Image Policies

```yaml
# Constraint Template for Image Security
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecureimages
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecureImages
      validation:
        properties:
          allowedRegistries:
            type: array
            items:
              type: string
          bannedTags:
            type: array
            items:
              type: string
  
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredsecureimages
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not starts_with(container.image, input.parameters.allowedRegistries[_])
        msg := sprintf("Image '%v' is not from allowed registry", [container.image])
      }
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        ends_with(container.image, input.parameters.bannedTags[_])
        msg := sprintf("Image '%v' uses banned tag", [container.image])
      }
```

```yaml
# Image Security Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecureImages
metadata:
  name: secure-images-only
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    - apiGroups: ["apps"]
      kinds: ["Deployment", "ReplicaSet", "DaemonSet", "StatefulSet"]
  
  parameters:
    allowedRegistries:
    - "registry.company.com/"
    - "gcr.io/company-project/"
    - "docker.io/library/"
    
    bannedTags:
    - ":latest"
    - ":master"
    - ":main"
```

---

## 🔹 Admission Controllers

### ValidatingAdmissionWebhook

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: security-validator
webhooks:
- name: pod-security.example.com
  clientConfig:
    service:
      name: security-webhook
      namespace: security-system
      path: /validate
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
  
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  failurePolicy: Fail
```

### MutatingAdmissionWebhook

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionWebhook
metadata:
  name: security-mutator
webhooks:
- name: pod-security-mutator.example.com
  clientConfig:
    service:
      name: security-webhook
      namespace: security-system
      path: /mutate
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
  
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail
```

### OPA Gatekeeper Policies

```yaml
# Security Constraint Template
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityContext
      validation:
        properties:
          runAsNonRoot:
            type: boolean
          readOnlyRootFilesystem:
            type: boolean
  
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredsecuritycontext
      
      violation[{"msg": msg}] {
        input.parameters.runAsNonRoot
        container := input.review.object.spec.containers[_]
        not container.securityContext.runAsNonRoot
        msg := "Container must run as non-root user"
      }
      
      violation[{"msg": msg}] {
        input.parameters.readOnlyRootFilesystem
        container := input.review.object.spec.containers[_]
        not container.securityContext.readOnlyRootFilesystem
        msg := "Container must have read-only root filesystem"
      }
```

```yaml
# Security Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityContext
metadata:
  name: must-have-security-context
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  
  parameters:
    runAsNonRoot: true
    readOnlyRootFilesystem: true
```

---

## 🔹 Service Mesh Security

### Istio Security Configuration

```yaml
# PeerAuthentication - mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  # Require mTLS for all workloads
  mtls:
    mode: STRICT
```

```yaml
# AuthorizationPolicy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: frontend-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: frontend
  
  rules:
  # Allow GET requests from authenticated users
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/web-sa"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/*"]
  
  # Allow POST requests with JWT token
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/data"]
    when:
    - key: request.headers[authorization]
      values: ["Bearer *"]
```

```yaml
# RequestAuthentication - JWT
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-server
  
  jwtRules:
  - issuer: "https://auth.example.com"
    jwksUri: "https://auth.example.com/.well-known/jwks.json"
    audiences:
    - "api.example.com"
    forwardOriginalToken: true
```

---

## 🔹 Security Monitoring

### Falco Security Rules

```yaml
# Falco ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-config
  namespace: falco-system
data:
  falco.yaml: |
    rules_file:
      - /etc/falco/falco_rules.yaml
      - /etc/falco/k8s_audit_rules.yaml
      - /etc/falco/rules.d
    
    json_output: true
    json_include_output_property: true
    
    http_output:
      enabled: true
      url: "http://falcosidekick:2801/"
  
  custom_rules.yaml: |
    - rule: Suspicious Container Activity
      desc: Detect suspicious container activities
      condition: >
        spawned_process and container and
        (proc.name in (nc, ncat, netcat, nmap, dig, nslookup, tcpdump))
      output: >
        Suspicious network tool launched in container
        (user=%user.name command=%proc.cmdline container=%container.name
        image=%container.image.repository:%container.image.tag)
      priority: WARNING
      tags: [network, mitre_discovery]
    
    - rule: Privileged Container Spawned
      desc: Detect privileged container creation
      condition: >
        k8s_audit and ka.verb=create and ka.target.resource=pods and
        ka.req.pod.containers.privileged intersects (true)
      output: >
        Privileged container created
        (user=%ka.user.name pod=%ka.target.name ns=%ka.target.namespace
        image=%ka.req.pod.containers.image)
      priority: HIGH
      tags: [k8s, privileged]
```

### Security Scanning with Trivy

```yaml
# Trivy Operator Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: trivy-operator
  namespace: trivy-system
data:
  trivy.repository: "ghcr.io/aquasecurity/trivy"
  trivy.tag: "0.35.0"
  
  # Vulnerability scanning
  vulnerabilityReports.scanner: "Trivy"
  vulnerabilityReports.scanJobTimeout: "5m"
  
  # Configuration audit
  configAuditReports.scanner: "Trivy"
  
  # Exposed secrets
  exposedSecretReports.scanner: "Trivy"
  
  # RBAC assessment
  rbacAssessmentReports.scanner: "Trivy"
```

---

## 🔹 Best Practices

### Security Checklist

```yaml
# Cluster Security:
✓ Enable RBAC
✓ Use Pod Security Standards
✓ Configure Network Policies
✓ Enable audit logging
✓ Secure etcd with TLS
✓ Use admission controllers
✓ Regular security updates

# Pod Security:
✓ Run as non-root user
✓ Use read-only root filesystem
✓ Drop all capabilities
✓ Set resource limits
✓ Use security contexts
✓ Avoid privileged containers
✓ Use specific image tags

# Network Security:
✓ Implement network segmentation
✓ Use service mesh for mTLS
✓ Secure ingress with TLS
✓ Monitor network traffic
✓ Use private registries
✓ Scan images for vulnerabilities

# Secrets Management:
✓ Use external secret management
✓ Rotate secrets regularly
✓ Encrypt secrets at rest
✓ Limit secret access
✓ Avoid hardcoded secrets
✓ Use service accounts properly
```

### Security Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
  labels:
    app: secure-app
    version: v1.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  
  template:
    metadata:
      labels:
        app: secure-app
        version: v1.0
      annotations:
        # Security annotations
        container.apparmor.security.beta.kubernetes.io/app: runtime/default
    
    spec:
      # Service account with minimal permissions
      serviceAccountName: secure-app-sa
      automountServiceAccountToken: true
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      # Image pull secrets
      imagePullSecrets:
      - name: registry-secret
      
      containers:
      - name: app
        image: registry.company.com/secure-app:v1.0.0
        imagePullPolicy: Always
        
        # Container security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop: ["ALL"]
          seccompProfile:
            type: RuntimeDefault
        
        # Resource limits
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTPS
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
            scheme: HTTPS
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # Environment variables from secrets
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        
        # Volume mounts
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      - name: tls-certs
        secret:
          secretName: app-tls-secret
          defaultMode: 0400
```

---

## 🔹 Interview Questions

### Basic Security Questions

**Q: What are the main security layers in Kubernetes?**
A: The 4C's of Cloud Native Security:
- **Cloud**: Infrastructure security (AWS, GCP, Azure)
- **Cluster**: Kubernetes cluster security (RBAC, network policies)
- **Container**: Container runtime security (security contexts, image scanning)
- **Code**: Application code security (secrets management, secure coding)

**Q: Explain RBAC in Kubernetes.**
A: Role-Based Access Control (RBAC) controls access to Kubernetes resources:
- **ServiceAccount**: Identity for pods
- **Role/ClusterRole**: Define permissions
- **RoleBinding/ClusterRoleBinding**: Bind roles to subjects
- **Subjects**: Users, groups, or service accounts

**Q: What is a Security Context?**
A: Security Context defines privilege and access control settings for pods/containers:
- **runAsUser/runAsGroup**: User/group ID
- **runAsNonRoot**: Prevent root execution
- **readOnlyRootFilesystem**: Make filesystem read-only
- **capabilities**: Linux capabilities management
- **seccompProfile**: Seccomp security profiles

### Advanced Security Questions

**Q: How do you implement network segmentation in Kubernetes?**
A: Using Network Policies:
- **Default deny**: Block all traffic by default
- **Ingress rules**: Control incoming traffic
- **Egress rules**: Control outgoing traffic
- **Label selectors**: Target specific pods/namespaces
- **Service mesh**: Additional layer with mTLS

**Q: Explain Pod Security Standards.**
A: Three security profiles:
- **Privileged**: Unrestricted policy (default)
- **Baseline**: Minimally restrictive, prevents known privilege escalations
- **Restricted**: Heavily restricted, follows pod hardening best practices

**Q: How do you secure secrets in Kubernetes?**
A: Multiple approaches:
- **Kubernetes Secrets**: Base64 encoded, encrypted at rest
- **External Secret Operators**: AWS Secrets Manager, HashiCorp Vault
- **Secret rotation**: Automated secret updates
- **RBAC**: Limit secret access
- **Encryption**: Enable encryption at rest for etcd

### Troubleshooting Questions

**Q: Pod fails with "container has runAsNonRoot and image will run as root"**
A: Solutions:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000  # Specify non-root user
```

**Q: Network Policy blocks legitimate traffic**
A: Debug steps:
1. Check policy selectors match intended pods
2. Verify ingress/egress rules
3. Test with temporary allow-all policy
4. Use network policy testing tools

**Q: Image pull fails with authentication error**
A: Check:
1. Image pull secrets are correctly configured
2. Secret contains valid registry credentials
3. Service account references the pull secret
4. Registry permissions are correct

This comprehensive security guide covers all essential Kubernetes security aspects for senior DevOps engineer interviews, including practical examples, manifest configurations, and real-world troubleshooting scenarios.
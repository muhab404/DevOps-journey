# 🔐 Kubernetes ConfigMaps & Secrets - DevOps Interview Guide

## Table of Contents
1. [Overview](#overview)
2. [ConfigMaps](#configmaps)
3. [Secrets](#secrets)
4. [Usage Patterns](#usage-patterns)
5. [Security Considerations](#security-considerations)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Interview Questions](#interview-questions)

---

## 🔹 Overview

### ConfigMaps vs Secrets

| Aspect | ConfigMaps | Secrets |
|--------|------------|---------|
| **Purpose** | Non-sensitive configuration | Sensitive data |
| **Encoding** | Plain text | Base64 encoded |
| **Storage** | etcd (plain text) | etcd (base64) |
| **Size Limit** | 1 MiB | 1 MiB |
| **Use Cases** | App config, environment variables | Passwords, tokens, certificates |

### Data Injection Methods

```yaml
# Three ways to consume ConfigMaps/Secrets:
1. Environment Variables - Individual key-value pairs
2. Volume Mounts - Files in filesystem
3. Command Line Arguments - Via environment variables
```

---

## 🔹 ConfigMaps

### Creating ConfigMaps

#### Imperative Creation

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=database_host=postgres.example.com \
  --from-literal=database_port=5432 \
  --from-literal=debug_mode=true

# From files
kubectl create configmap nginx-config --from-file=nginx.conf

# From directory
kubectl create configmap app-configs --from-file=config/

# From environment file
kubectl create configmap env-config --from-env-file=app.env
```

#### Declarative Creation

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: web-application
    component: configuration
data:
  # Simple key-value pairs
  database_host: "postgres.example.com"
  database_port: "5432"
  debug_mode: "false"
  log_level: "info"
  
  # Multi-line configuration
  app.properties: |
    server.port=8080
    server.servlet.context-path=/api
    spring.datasource.url=jdbc:postgresql://postgres:5432/mydb
    spring.datasource.username=admin
    logging.level.com.example=DEBUG
  
  # JSON configuration
  config.json: |
    {
      "server": {
        "port": 8080,
        "host": "0.0.0.0"
      },
      "database": {
        "host": "postgres.example.com",
        "port": 5432,
        "name": "myapp"
      },
      "features": {
        "authentication": true,
        "logging": true
      }
    }
  
  # YAML configuration
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    database:
      host: postgres.example.com
      port: 5432
      name: myapp
    features:
      authentication: true
      logging: true
```

### Using ConfigMaps as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-env-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    env:
    # Single environment variable from ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host
    
    # All keys from ConfigMap as environment variables
    envFrom:
    - configMapRef:
        name: app-config
    
    # Prefix all environment variables
    envFrom:
    - configMapRef:
        name: app-config
      prefix: "APP_"
```

### Using ConfigMaps as Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    # Mount entire ConfigMap
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
    
    # Mount specific keys
    - name: app-properties
      mountPath: /etc/app
      readOnly: true
  
  volumes:
  # Mount all keys from ConfigMap
  - name: config-volume
    configMap:
      name: app-config
  
  # Mount specific keys with custom names
  - name: app-properties
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: application.properties
      - key: config.json
        path: app-config.json
      defaultMode: 0644
```

---

## 🔹 Secrets

### Secret Types

```yaml
# Built-in Secret Types:
Opaque: Generic secret (default)
kubernetes.io/service-account-token: Service account token
kubernetes.io/dockercfg: Docker registry credentials (legacy)
kubernetes.io/dockerconfigjson: Docker registry credentials
kubernetes.io/basic-auth: Basic authentication credentials
kubernetes.io/ssh-auth: SSH authentication credentials
kubernetes.io/tls: TLS certificate and key
```

### Creating Secrets

#### Imperative Creation

```bash
# Generic secret from literals
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secretpassword

# TLS secret from files
kubectl create secret tls tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key

# Docker registry secret
kubectl create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com

# From files
kubectl create secret generic app-secrets --from-file=secrets/
```

#### Declarative Creation

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
  labels:
    app: database
    component: authentication
type: Opaque
data:
  # Base64 encoded values
  username: YWRtaW4=        # admin
  password: c2VjcmV0cGFzcw==  # secretpass
  database: bXlhcHA=        # myapp

---
# String data (automatically base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
type: Opaque
stringData:
  api_key: "sk-1234567890abcdef"
  webhook_secret: "whsec_abcdef123456"
  jwt_secret: "super-secret-jwt-key"

---
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
# Docker Registry Secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5leGFtcGxlLmNvbSI6eyJ1c2VybmFtZSI6InVzZXIiLCJwYXNzd29yZCI6InBhc3N3b3JkIiwiYXV0aCI6ImRYTmxjanB3WVhOemQyOXlaQT09In19fQ==
```

### Using Secrets as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    env:
    # Single environment variable from Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    
    # All keys from Secret as environment variables
    envFrom:
    - secretRef:
        name: db-credentials
    
    # Optional secret (won't fail if missing)
    env:
    - name: OPTIONAL_KEY
      valueFrom:
        secretKeyRef:
          name: optional-secret
          key: optional-key
          optional: true
```

### Using Secrets as Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    # Mount entire Secret
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
    
    # Mount TLS certificates
    - name: tls-certs
      mountPath: /etc/ssl/certs
      readOnly: true
  
  volumes:
  # Mount all keys from Secret
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400  # Read-only for owner
  
  # Mount specific keys with custom names
  - name: tls-certs
    secret:
      secretName: tls-secret
      items:
      - key: tls.crt
        path: server.crt
        mode: 0444
      - key: tls.key
        path: server.key
        mode: 0400
```

---

## 🔹 Usage Patterns

### Environment Variable Injection

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
        env:
        # ConfigMap values
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log_level
        
        # Secret values
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        
        # Direct values
        - name: APP_ENV
          value: "production"
        
        # Inject all ConfigMap keys
        envFrom:
        - configMapRef:
            name: app-config
        
        # Inject all Secret keys
        envFrom:
        - secretRef:
            name: api-keys
```

### File-based Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        # Nginx configuration
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
        
        # SSL certificates
        - name: ssl-certs
          mountPath: /etc/ssl/certs
          readOnly: true
        
        # Application configuration
        - name: app-config
          mountPath: /etc/app/config
          readOnly: true
      
      volumes:
      # Nginx configuration from ConfigMap
      - name: nginx-config
        configMap:
          name: nginx-config
          items:
          - key: nginx.conf
            path: nginx.conf
      
      # SSL certificates from Secret
      - name: ssl-certs
        secret:
          secretName: tls-secret
          defaultMode: 0444
      
      # Application configuration from ConfigMap
      - name: app-config
        configMap:
          name: app-config
```

---

## 🔹 Security Considerations

### Secret Security Best Practices

```yaml
# 1. Use RBAC to restrict access
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["app-secrets"]  # Specific secrets only

---
# 2. Use service accounts with minimal permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
---
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: app-service-account
  automountServiceAccountToken: false  # Disable if not needed
```

### Encryption at Rest

```yaml
# Enable encryption at rest in etcd
apiVersion: apiserver.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

### Secret Rotation

```bash
# Update secret with new values
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=newpassword \
  --dry-run=client -o yaml | kubectl apply -f -

# Rolling restart to pick up new secrets
kubectl rollout restart deployment/web-app
```

---

## 🔹 Practical Examples

### 1. Complete Web Application Configuration

```yaml
# Application ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
  namespace: production
data:
  # Environment configuration
  environment: "production"
  log_level: "info"
  debug_mode: "false"
  
  # Database configuration
  database_host: "postgres.production.svc.cluster.local"
  database_port: "5432"
  database_name: "webapp"
  
  # Redis configuration
  redis_host: "redis.production.svc.cluster.local"
  redis_port: "6379"
  
  # Application properties
  application.properties: |
    server.port=8080
    server.servlet.context-path=/api/v1
    
    # Database settings
    spring.datasource.url=jdbc:postgresql://${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}
    spring.datasource.username=${DATABASE_USERNAME}
    spring.datasource.password=${DATABASE_PASSWORD}
    
    # Redis settings
    spring.redis.host=${REDIS_HOST}
    spring.redis.port=${REDIS_PORT}
    spring.redis.password=${REDIS_PASSWORD}
    
    # Logging
    logging.level.com.example=${LOG_LEVEL}
    logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n

---
# Application Secrets
apiVersion: v1
kind: Secret
metadata:
  name: web-app-secrets
  namespace: production
type: Opaque
stringData:
  database_username: "webapp_user"
  database_password: "super_secure_db_password"
  redis_password: "redis_secure_password"
  jwt_secret: "jwt-signing-secret-key"
  api_key: "external-api-key-12345"

---
# Deployment using ConfigMap and Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
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
        image: web-app:v2.0
        ports:
        - containerPort: 8080
        
        # Environment variables from ConfigMap
        env:
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: environment
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: log_level
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: database_host
        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: database_port
        - name: DATABASE_NAME
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: database_name
        
        # Environment variables from Secret
        - name: DATABASE_USERNAME
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: database_username
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: database_password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: jwt_secret
        
        # Mount configuration file
        volumeMounts:
        - name: app-config
          mountPath: /app/config/application.properties
          subPath: application.properties
          readOnly: true
        
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
      
      volumes:
      - name: app-config
        configMap:
          name: web-app-config
          items:
          - key: application.properties
            path: application.properties
```

### 2. Database with Configuration and Secrets

```yaml
# Database ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  # PostgreSQL configuration
  postgresql.conf: |
    # Connection settings
    listen_addresses = '*'
    port = 5432
    max_connections = 100
    
    # Memory settings
    shared_buffers = 256MB
    effective_cache_size = 1GB
    work_mem = 4MB
    
    # Logging
    log_statement = 'all'
    log_duration = on
    log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
  
  # Initialization script
  init.sql: |
    CREATE DATABASE webapp;
    CREATE USER webapp_user WITH PASSWORD 'temp_password';
    GRANT ALL PRIVILEGES ON DATABASE webapp TO webapp_user;

---
# Database Secrets
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
type: Opaque
stringData:
  postgres_user: "postgres"
  postgres_password: "super_secure_postgres_password"
  webapp_user: "webapp_user"
  webapp_password: "webapp_secure_password"

---
# PostgreSQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
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
        ports:
        - containerPort: 5432
        
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: postgres_user
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: postgres_password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        
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
        
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
      
      volumes:
      - name: postgres-config
        configMap:
          name: postgres-config
          items:
          - key: postgresql.conf
            path: postgresql.conf
      - name: postgres-init
        configMap:
          name: postgres-config
          items:
          - key: init.sql
            path: init.sql
  
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 3. Nginx with SSL Configuration

```yaml
# Nginx ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    
    http {
        upstream backend {
            server web-app:8080;
        }
        
        server {
            listen 80;
            server_name example.com;
            return 301 https://$server_name$request_uri;
        }
        
        server {
            listen 443 ssl http2;
            server_name example.com;
            
            ssl_certificate /etc/ssl/certs/tls.crt;
            ssl_certificate_key /etc/ssl/private/tls.key;
            ssl_protocols TLSv1.2 TLSv1.3;
            ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
            
            location / {
                proxy_pass http://backend;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
    }

---
# TLS Secret (create with actual certificates)
apiVersion: v1
kind: Secret
metadata:
  name: nginx-tls
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # Base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi... # Base64 encoded private key

---
# Nginx Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        - containerPort: 443
        
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
        - name: tls-certs
          mountPath: /etc/ssl/certs
          readOnly: true
        - name: tls-private
          mountPath: /etc/ssl/private
          readOnly: true
        
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
      
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: tls-certs
        secret:
          secretName: nginx-tls
          items:
          - key: tls.crt
            path: tls.crt
            mode: 0444
      - name: tls-private
        secret:
          secretName: nginx-tls
          items:
          - key: tls.key
            path: tls.key
            mode: 0400
```

---

## 🔹 Best Practices

### 1. Configuration Management

```yaml
# Separate ConfigMaps by environment
ConfigMaps:
  - app-config-dev
  - app-config-staging  
  - app-config-production

# Use consistent naming conventions
Naming Pattern:
  - <app-name>-<component>-<environment>
  - web-app-config-production
  - web-app-secrets-production
```

### 2. Secret Management

```yaml
# Use external secret management
External Tools:
  - HashiCorp Vault
  - AWS Secrets Manager
  - Azure Key Vault
  - Google Secret Manager

# Implement secret rotation
Rotation Strategy:
  - Automated rotation schedules
  - Zero-downtime updates
  - Rollback capabilities
```

### 3. Security Hardening

```yaml
# Minimize secret exposure
Security Measures:
  - Use volume mounts instead of environment variables
  - Set restrictive file permissions (0400 for secrets)
  - Implement RBAC for secret access
  - Enable audit logging
  - Use service mesh for secret injection
```

### 4. Operational Excellence

```yaml
# Monitoring and Alerting
Monitoring:
  - ConfigMap/Secret change events
  - Failed secret mounts
  - Permission denied errors
  - Secret rotation status

# Documentation
Documentation:
  - Configuration schema
  - Secret rotation procedures
  - Emergency access procedures
  - Troubleshooting guides
```

---

## 🔹 Troubleshooting

### Common Issues

#### 1. ConfigMap/Secret Not Found

```bash
# Check if ConfigMap exists
kubectl get configmap <name> -n <namespace>

# Check if Secret exists
kubectl get secret <name> -n <namespace>

# Describe for more details
kubectl describe configmap <name>
kubectl describe secret <name>
```

#### 2. Permission Denied

```bash
# Check RBAC permissions
kubectl auth can-i get configmaps --as=system:serviceaccount:<namespace>:<sa-name>
kubectl auth can-i get secrets --as=system:serviceaccount:<namespace>:<sa-name>

# Check service account
kubectl get serviceaccount <sa-name>
kubectl describe serviceaccount <sa-name>
```

#### 3. Mount Issues

```bash
# Check pod events
kubectl describe pod <pod-name>

# Check volume mounts
kubectl get pod <pod-name> -o yaml | grep -A 10 volumeMounts

# Check if files are mounted correctly
kubectl exec -it <pod-name> -- ls -la /path/to/mount
```

### Debugging Commands

```bash
# List all ConfigMaps and Secrets
kubectl get configmaps,secrets -A

# View ConfigMap/Secret content
kubectl get configmap <name> -o yaml
kubectl get secret <name> -o yaml

# Decode secret values
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d

# Check pod environment variables
kubectl exec -it <pod-name> -- env | grep <variable>

# Check mounted files
kubectl exec -it <pod-name> -- cat /path/to/config/file
```

---

## 🔹 Interview Questions

### Conceptual Questions

**Q1: What's the difference between ConfigMaps and Secrets?**
- **ConfigMaps**: Non-sensitive configuration data, stored as plain text
- **Secrets**: Sensitive data, base64 encoded, additional security features

**Q2: How are Secrets stored in etcd?**
- Base64 encoded (not encrypted by default)
- Can enable encryption at rest
- Transmitted over TLS

**Q3: What are the ways to consume ConfigMaps and Secrets?**
- Environment variables
- Volume mounts
- Command line arguments (via env vars)

### Technical Questions

**Q4: How do you update a ConfigMap without restarting pods?**
```bash
# ConfigMaps mounted as volumes update automatically
# Environment variables require pod restart
kubectl rollout restart deployment/<name>
```

**Q5: What's the size limit for ConfigMaps and Secrets?**
- Both have 1 MiB limit
- Consider using external storage for larger configurations

**Q6: How do you handle secret rotation?**
```yaml
# Update secret, then rolling restart
kubectl create secret generic <name> --from-literal=key=newvalue --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deployment/<name>
```

### Scenario-Based Questions

**Q7: Design configuration management for a multi-environment application.**
```yaml
# Separate ConfigMaps per environment
# Use Kustomize or Helm for templating
# External secret management integration
# Automated deployment pipelines
```

**Q8: How would you secure database credentials?**
```yaml
# Use Secrets with restrictive RBAC
# Mount as volumes with proper permissions
# Implement secret rotation
# Use external secret managers
# Enable audit logging
```

**Q9: Troubleshoot a pod that can't access its ConfigMap.**
```bash
# Check ConfigMap exists in correct namespace
# Verify RBAC permissions
# Check pod service account
# Verify volume mount configuration
# Check for typos in names/keys
```

### Best Practices Questions

**Q10: What are ConfigMap and Secret best practices?**
- Use descriptive names and labels
- Implement proper RBAC
- Use volume mounts for secrets
- Enable encryption at rest
- Implement secret rotation
- Monitor configuration changes
- Use external secret management for production
# Helm Comprehensive Course Guide

## 1. Getting Started with Helm

### Before Helm
**The Problem:**
- Manual YAML management for complex applications
- No versioning or rollback capabilities
- Difficult environment-specific configurations
- Copy-paste errors and inconsistencies
- No dependency management

**Pain Points:**
```bash
# Manual deployment process
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f ingress.yaml
# Repeat for each environment with different values
```

### What is Helm
**Definition:** Kubernetes package manager that uses templates to generate manifests

**Key Components:**
- **Chart**: Package of Kubernetes resources
- **Release**: Instance of a chart deployed to cluster
- **Repository**: Collection of charts
- **Values**: Configuration parameters

### After Helm
**Benefits Achieved:**
```bash
# Simple deployment
helm install myapp ./mychart

# Easy upgrades
helm upgrade myapp ./mychart --set image.tag=v2.0

# Quick rollbacks
helm rollback myapp 1
```

### Charts and Repositories
**Chart Structure:**
```
mychart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl
└── charts/
```

**Repository Types:**
- HTTP/HTTPS repositories
- OCI registries
- Local file system
- Git repositories

### Installing Helm
**Installation Methods:**
```bash
# Script-based (Linux/macOS)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Package managers
brew install helm                    # macOS
choco install kubernetes-helm        # Windows
snap install helm --classic          # Linux

# Binary download
wget https://get.helm.sh/helm-v3.x.x-linux-amd64.tar.gz
```

### Working with Chart Repositories
**Repository Management:**
```bash
# Add repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# Update repository index
helm repo update

# List repositories
helm repo list

# Search charts
helm search repo nginx

# Remove repository
helm repo remove bitnami
```

### The Magic of Helm
**Template Example:**
```yaml
# Before Helm (static)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3

# After Helm (dynamic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
```

## 2. Helm Basics & Core Commands

### Using the Same Installation Name
**Release Management:**
```bash
# Install with specific name
helm install myapp bitnami/nginx

# Upgrade existing release
helm upgrade myapp bitnami/nginx

# Install or upgrade (idempotent)
helm upgrade --install myapp bitnami/nginx
```

### Listing and Uninstalling Releases
**Release Operations:**
```bash
# List all releases
helm list
helm list --all-namespaces

# List releases in specific namespace
helm list -n production

# Uninstall release
helm uninstall myapp

# Uninstall but keep history
helm uninstall myapp --keep-history
```

### Providing Custom Values
**Value Override Methods:**
```bash
# Using --set flag
helm install myapp bitnami/nginx --set replicaCount=5

# Using values file
helm install myapp bitnami/nginx -f custom-values.yaml

# Multiple values files (precedence: right to left)
helm install myapp bitnami/nginx -f base-values.yaml -f env-values.yaml

# Combining methods
helm install myapp bitnami/nginx -f values.yaml --set image.tag=latest
```

**Custom Values File:**
```yaml
# custom-values.yaml
replicaCount: 3
image:
  tag: "1.21"
service:
  type: LoadBalancer
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Helm Upgrade & Rollback
**Upgrade Operations:**
```bash
# Basic upgrade
helm upgrade myapp bitnami/nginx

# Upgrade with new values
helm upgrade myapp bitnami/nginx --set replicaCount=5

# Force upgrade
helm upgrade myapp bitnami/nginx --force

# Rollback to previous version
helm rollback myapp

# Rollback to specific revision
helm rollback myapp 2
```

### Release Records & History
**History Management:**
```bash
# View release history
helm history myapp

# Detailed history
helm history myapp --max 10

# Get release information
helm get all myapp
helm get values myapp
helm get manifest myapp
helm get notes myapp
```

### Helm Dry-Run and Template
**Testing and Debugging:**
```bash
# Dry run installation
helm install myapp bitnami/nginx --dry-run

# Generate templates locally
helm template myapp bitnami/nginx

# Debug template rendering
helm template myapp bitnami/nginx --debug

# Validate templates
helm template myapp ./mychart --validate
```

### Helm Get (resources, manifest)
**Resource Inspection:**
```bash
# Get all release information
helm get all myapp

# Get computed values
helm get values myapp

# Get rendered manifest
helm get manifest myapp

# Get release notes
helm get notes myapp

# Get hooks
helm get hooks myapp
```

### Generating Release Names
**Name Generation:**
```bash
# Auto-generate release name
helm install bitnami/nginx --generate-name

# Custom name prefix
helm install bitnami/nginx --generate-name --name-template "myapp-{{randAlpha 6 | lower}}"
```

### Namespaces, Wait & Timeout, Atomic Installs
**Advanced Installation Options:**
```bash
# Install in specific namespace
helm install myapp bitnami/nginx -n production --create-namespace

# Wait for resources to be ready
helm install myapp bitnami/nginx --wait --timeout=300s

# Atomic installation (rollback on failure)
helm install myapp bitnami/nginx --atomic

# Combine options
helm install myapp bitnami/nginx -n prod --create-namespace --wait --atomic
```

### Forceful Upgrades & Cleanup on Failed Updates
**Error Recovery:**
```bash
# Force upgrade (recreate resources)
helm upgrade myapp bitnami/nginx --force

# Reset values to chart defaults
helm upgrade myapp bitnami/nginx --reset-values

# Cleanup on failed upgrade
helm upgrade myapp bitnami/nginx --cleanup-on-fail

# Replace existing installation
helm install myapp bitnami/nginx --replace
```

## 3. Creating and Managing Charts

### Creating the First Chart
**Chart Creation:**
```bash
# Create new chart
helm create webapp

# Chart structure created
webapp/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── _helpers.tpl
└── charts/
```

### Installing a Chart
**Local Chart Installation:**
```bash
# Install from local directory
helm install myapp ./webapp

# Install with custom values
helm install myapp ./webapp -f production-values.yaml

# Debug installation
helm install myapp ./webapp --dry-run --debug
```

### Chart.yaml Explained
**Chart Metadata:**
```yaml
apiVersion: v2
name: webapp
description: A web application Helm chart
type: application
version: 0.1.0
appVersion: "1.0"
home: https://example.com
sources:
  - https://github.com/example/webapp
maintainers:
  - name: John Doe
    email: john@example.com
keywords:
  - web
  - application
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### Templates in Brief
**Template Basics:**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        ports:
        - containerPort: {{ .Values.service.port }}
```

### Values.yaml
**Default Configuration:**
```yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

nodeSelector: {}
tolerations: []
affinity: {}
```

### Helpers File (_helpers.tpl)
**Template Helpers:**
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "webapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "webapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "webapp.labels" -}}
helm.sh/chart: {{ include "webapp.chart" . }}
{{ include "webapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

### Packaging and Linting Charts
**Chart Quality Control:**
```bash
# Lint chart for issues
helm lint ./webapp

# Package chart into archive
helm package ./webapp

# Package with specific version
helm package ./webapp --version 1.0.0

# Package and sign
helm package ./webapp --sign --key mykey

# Verify packaged chart
helm verify webapp-0.1.0.tgz
```

## 4. Template Deep Dive

### Template Actions & Information
**Built-in Objects:**
```yaml
# Release information
{{ .Release.Name }}
{{ .Release.Namespace }}
{{ .Release.Service }}

# Chart information
{{ .Chart.Name }}
{{ .Chart.Version }}
{{ .Chart.AppVersion }}

# Values
{{ .Values.replicaCount }}
{{ .Values.image.tag }}

# Files
{{ .Files.Get "config.yaml" }}

# Capabilities
{{ .Capabilities.KubeVersion }}
```

### Pipelines and Functions
**Template Functions:**
```yaml
# String functions
{{ .Values.name | upper }}
{{ .Values.name | lower }}
{{ .Values.name | title }}
{{ .Values.name | quote }}

# Default values
{{ .Values.tag | default "latest" }}

# Type conversion
{{ .Values.port | toString }}
{{ .Values.enabled | toString }}

# Date functions
{{ now | date "2006-01-02" }}

# Encoding
{{ .Values.data | b64enc }}
{{ .Values.secret | b64dec }}
```

### Conditional Logic and "With"
**Control Structures:**
```yaml
# If conditions
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "webapp.fullname" . }}
{{- end }}

# If-else
{{- if eq .Values.service.type "NodePort" }}
  nodePort: {{ .Values.service.nodePort }}
{{- else }}
  port: {{ .Values.service.port }}
{{- end }}

# With (change scope)
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

### Variables, Loops, and Dict Iteration
**Advanced Templating:**
```yaml
# Variables
{{- $fullName := include "webapp.fullname" . }}
{{- $labels := include "webapp.labels" . }}

# Range over lists
{{- range .Values.hosts }}
- host: {{ . }}
{{- end }}

# Range over maps
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}

# Range with index
{{- range $index, $host := .Values.hosts }}
- host{{ $index }}: {{ $host }}
{{- end }}
```

### Debugging Templates
**Debug Techniques:**
```bash
# Render templates locally
helm template myapp ./webapp

# Debug with values
helm template myapp ./webapp --debug

# Specific template file
helm template myapp ./webapp -s templates/deployment.yaml

# Show computed values
helm template myapp ./webapp --debug | grep -A 20 "COMPUTED VALUES"
```

### Creating and Using Custom Templates
**Custom Template Definitions:**
```yaml
# In _helpers.tpl
{{- define "webapp.env" -}}
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
{{- end }}

# Usage in deployment.yaml
spec:
  containers:
  - name: webapp
    env:
    {{- include "webapp.env" . | nindent 4 }}
```

## 5. Advanced Charts & Dependencies

### Adding Dependencies
**Chart.yaml Dependencies:**
```yaml
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: https://charts.bitnami.com/bitnami
  - name: redis
    version: "^17.0.0"
    repository: https://charts.bitnami.com/bitnami
```

**Dependency Commands:**
```bash
# Download dependencies
helm dependency update

# List dependencies
helm dependency list

# Build dependency archive
helm dependency build
```

### Version Ranges & Repository Names
**Version Constraints:**
```yaml
dependencies:
  - name: postgresql
    version: "~11.1.0"      # >= 11.1.0, < 11.2.0
    repository: "@bitnami"
  - name: redis
    version: "^17.0.0"      # >= 17.0.0, < 18.0.0
    repository: "https://charts.bitnami.com/bitnami"
  - name: mongodb
    version: ">= 10.0.0"    # >= 10.0.0
    repository: "file://../mongodb"
```

### Conditional and Multiple Dependencies
**Conditional Dependencies:**
```yaml
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: mysql
    version: "9.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: mysql.enabled
    tags:
      - database
```

### Passing Values to Dependencies
**Values for Dependencies:**
```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    postgresPassword: "mypassword"
    database: "myapp"
  primary:
    persistence:
      enabled: true
      size: 8Gi

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      enabled: false
```

### Reading Values from Child Charts
**Accessing Child Values:**
```yaml
# In parent templates
{{- if .Values.postgresql.enabled }}
- name: DATABASE_URL
  value: "postgresql://{{ .Values.postgresql.auth.username }}@{{ include "postgresql.primary.fullname" .Subcharts.postgresql }}"
{{- end }}
```

### Using Non-Exported Values
**Global Values:**
```yaml
# values.yaml
global:
  imageRegistry: "myregistry.com"
  storageClass: "fast-ssd"

# Available in all charts
image: "{{ .Values.global.imageRegistry }}/myapp:latest"
```

### Hooks (Intro & Creating Hooks)
**Hook Types:**
- pre-install, post-install
- pre-delete, post-delete
- pre-upgrade, post-upgrade
- pre-rollback, post-rollback

**Hook Example:**
```yaml
# templates/pre-install-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "webapp.fullname" . }}-pre-install
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: pre-install
        image: busybox
        command: ['sh', '-c', 'echo "Pre-install hook executed"']
      restartPolicy: Never
```

### Chart Testing
**Test Templates:**
```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "webapp.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "webapp.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

**Running Tests:**
```bash
# Run chart tests
helm test myapp

# Run tests with logs
helm test myapp --logs
```

## 6. Repositories

### Setting up a Local Repository
**Local HTTP Server:**
```bash
# Create repository directory
mkdir helm-repo
cd helm-repo

# Package charts
helm package ../mychart

# Generate index
helm repo index . --url http://localhost:8080

# Start HTTP server
python3 -m http.server 8080
```

### Hosting on a Web Server
**Repository Structure:**
```
helm-repo/
├── index.yaml
├── mychart-0.1.0.tgz
├── mychart-0.2.0.tgz
└── anotherchart-1.0.0.tgz
```

**Index.yaml:**
```yaml
apiVersion: v1
entries:
  mychart:
  - apiVersion: v2
    created: "2023-01-01T00:00:00Z"
    description: My application chart
    digest: sha256:abc123...
    name: mychart
    urls:
    - https://myrepo.com/charts/mychart-0.1.0.tgz
    version: 0.1.0
```

### Using the Repository & GitHub Pages
**GitHub Pages Setup:**
```bash
# Create gh-pages branch
git checkout --orphan gh-pages

# Add charts and index
helm package ../mychart
helm repo index . --url https://username.github.io/helm-charts

# Commit and push
git add .
git commit -m "Initial chart repository"
git push origin gh-pages
```

### Helm Pull & Updating Repositories
**Repository Operations:**
```bash
# Pull chart without installing
helm pull bitnami/nginx

# Pull specific version
helm pull bitnami/nginx --version 13.2.0

# Pull and untar
helm pull bitnami/nginx --untar

# Update repository indexes
helm repo update

# Search updated repositories
helm search repo nginx --versions
```

### OCI Repositories (Experimental & Stable Usage)
**OCI Registry Support:**
```bash
# Enable OCI support (Helm 3.8+)
export HELM_EXPERIMENTAL_OCI=1

# Login to registry
helm registry login myregistry.com

# Push chart to OCI registry
helm push mychart-0.1.0.tgz oci://myregistry.com/helm-charts

# Install from OCI registry
helm install myapp oci://myregistry.com/helm-charts/mychart --version 0.1.0
```

## 7. Chart Security

### Generating PGP Keys
**Key Generation:**
```bash
# Generate GPG key
gpg --full-generate-key

# List keys
gpg --list-keys

# Export public key
gpg --armor --export user@example.com > public.key

# Export private key
gpg --armor --export-secret-keys user@example.com > private.key
```

### Signing and Verifying Charts
**Chart Signing:**
```bash
# Sign chart during packaging
helm package mychart --sign --key 'John Doe' --keyring ~/.gnupg/secring.gpg

# Verify signed chart
helm verify mychart-0.1.0.tgz

# Install with verification
helm install myapp mychart-0.1.0.tgz --verify
```

### Verifying During Install
**Verification Setup:**
```bash
# Add public key to keyring
gpg --import public.key

# Install with verification
helm install myapp ./mychart --verify --keyring ~/.gnupg/pubring.gpg
```

## 8. Real-World Use Case

### Creating and Updating a Chart
**Microservice Chart:**
```bash
# Create chart for microservice
helm create user-service

# Customize for microservice
# - Update Chart.yaml with proper metadata
# - Configure values.yaml for service requirements
# - Modify templates for specific needs
```

### Updating Deployment & Service
**Production-Ready Templates:**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "user-service.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        livenessProbe:
          httpGet:
            path: /health
            port: {{ .Values.service.port }}
        readinessProbe:
          httpGet:
            path: /ready
            port: {{ .Values.service.port }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

### Adding Dependencies & ConfigMaps
**Database Integration:**
```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled

# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "user-service.fullname" . }}-config
data:
  database.url: {{ .Values.database.url | quote }}
  redis.url: {{ .Values.redis.url | quote }}
```

### Troubleshooting & Testing a Release
**Debug Process:**
```bash
# Check template rendering
helm template user-service ./user-service --debug

# Dry run installation
helm install user-service ./user-service --dry-run

# Install with debugging
helm install user-service ./user-service --debug --wait

# Check release status
helm status user-service

# View logs
kubectl logs -l app.kubernetes.io/name=user-service

# Run tests
helm test user-service
```

## 9. Helm Starters & Plugins

### Creating Starters & Using Them
**Starter Templates:**
```bash
# Create starter directory
mkdir -p ~/.helm/starters/microservice

# Copy template structure
cp -r mychart/* ~/.helm/starters/microservice/

# Create chart from starter
helm create newservice --starter microservice
```

### Installing and Using Plugins
**Popular Plugins:**
```bash
# Install diff plugin
helm plugin install https://github.com/databus23/helm-diff

# Use diff plugin
helm diff upgrade myapp ./mychart

# Install secrets plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# List installed plugins
helm plugin list
```

### Creating Custom Plugins
**Plugin Structure:**
```
helm-myplugin/
├── plugin.yaml
└── myplugin.sh
```

**plugin.yaml:**
```yaml
name: "myplugin"
version: "0.1.0"
usage: "Custom plugin for Helm"
description: "A custom Helm plugin"
command: "$HELM_PLUGIN_DIR/myplugin.sh"
```

## 10. Wrap-up

### Validating and Generating Schema
**JSON Schema Validation:**
```yaml
# values.schema.json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    },
    "image": {
      "type": "object",
      "properties": {
        "repository": {"type": "string"},
        "tag": {"type": "string"}
      },
      "required": ["repository"]
    }
  },
  "required": ["replicaCount", "image"]
}
```

### Final Notes & Best Practices

**Chart Development:**
- Use semantic versioning
- Include comprehensive documentation
- Implement proper testing
- Follow naming conventions
- Use resource limits and requests

**Security:**
- Never hardcode secrets
- Use RBAC appropriately
- Scan charts for vulnerabilities
- Sign charts for production use

**Operations:**
- Monitor release health
- Implement proper backup strategies
- Use namespaces for isolation
- Plan upgrade and rollback procedures

**Performance:**
- Optimize template rendering
- Use appropriate resource limits
- Consider chart size and complexity
- Implement efficient dependency management

This comprehensive course covers all aspects of Helm from basic concepts to advanced production usage, providing practical examples and real-world scenarios for mastering Kubernetes package management.
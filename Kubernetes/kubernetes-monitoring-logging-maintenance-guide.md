# 📊 Kubernetes Monitoring, Logging & Maintenance - DevOps Interview Guide

## Table of Contents
1. [Monitoring Overview](#monitoring-overview)
2. [Prometheus & Grafana](#prometheus--grafana)
3. [Logging Architecture](#logging-architecture)
4. [ELK/EFK Stack](#elkelk-stack)
5. [Health Checks & Probes](#health-checks--probes)
6. [Metrics & Alerting](#metrics--alerting)
7. [Cluster Maintenance](#cluster-maintenance)
8. [Backup & Recovery](#backup--recovery)
9. [Performance Tuning](#performance-tuning)
10. [Troubleshooting](#troubleshooting)
11. [Interview Questions](#interview-questions)

---

## 🔹 Monitoring Overview

### Monitoring Architecture

```yaml
# Monitoring Stack Components:
Metrics Collection:
  - Prometheus (metrics storage)
  - Node Exporter (node metrics)
  - kube-state-metrics (K8s object metrics)
  - cAdvisor (container metrics)

Visualization:
  - Grafana (dashboards)
  - Prometheus UI (queries)
  - Kubernetes Dashboard

Alerting:
  - Alertmanager (alert routing)
  - PagerDuty, Slack integration
  - Webhook notifications

Log Management:
  - Fluentd/Fluent Bit (log collection)
  - Elasticsearch (log storage)
  - Kibana (log visualization)
```

### Monitoring Namespace Setup

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring
    monitoring: enabled
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
  namespace: monitoring
automountServiceAccountToken: true
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-reader-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: monitoring-reader
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring
```

---

## 🔹 Prometheus & Grafana

### Prometheus Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: 'production'
        region: 'us-west-2'
    
    rule_files:
    - "/etc/prometheus/rules/*.yml"
    
    alerting:
      alertmanagers:
      - static_configs:
        - targets:
          - alertmanager:9093
    
    scrape_configs:
    # Prometheus itself
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
    
    # Kubernetes API server
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    
    # Kubernetes nodes
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics
    
    # Kubernetes pods
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
    
    # kube-state-metrics
    - job_name: 'kube-state-metrics'
      static_configs:
      - targets: ['kube-state-metrics:8080']
    
    # Node exporter
    - job_name: 'node-exporter'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_endpoints_name]
        regex: 'node-exporter'
        action: keep
```

### Prometheus Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: monitoring-sa
      containers:
      - name: prometheus
        image: prom/prometheus:v2.40.0
        args:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--storage.tsdb.path=/prometheus/'
        - '--web.console.libraries=/etc/prometheus/console_libraries'
        - '--web.console.templates=/etc/prometheus/consoles'
        - '--storage.tsdb.retention.time=15d'
        - '--web.enable-lifecycle'
        - '--web.enable-admin-api'
        ports:
        - containerPort: 9090
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus/
        - name: prometheus-storage
          mountPath: /prometheus/
        livenessProbe:
          httpGet:
            path: /-/healthy
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            path: /-/ready
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        persistentVolumeClaim:
          claimName: prometheus-pvc
```

### Grafana Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-config
  namespace: monitoring
data:
  grafana.ini: |
    [analytics]
    check_for_updates = true
    
    [grafana_net]
    url = https://grafana.net
    
    [log]
    mode = console
    
    [paths]
    data = /var/lib/grafana/data
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
    provisioning = /etc/grafana/provisioning
    
    [server]
    root_url = http://grafana.example.com/
    
    [security]
    admin_user = admin
    admin_password = $__env{GF_SECURITY_ADMIN_PASSWORD}
    
    [users]
    allow_sign_up = false
    
    [auth.anonymous]
    enabled = false
  
  datasources.yml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus:9090
      isDefault: true
      editable: true
    - name: Loki
      type: loki
      access: proxy
      url: http://loki:3100
      editable: true
  
  dashboards.yml: |
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      updateIntervalSeconds: 10
      allowUiUpdates: true
      options:
        path: /var/lib/grafana/dashboards
```

### Grafana Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:9.3.0
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-secret
              key: admin-password
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        volumeMounts:
        - name: grafana-config
          mountPath: /etc/grafana/
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-dashboards
          mountPath: /var/lib/grafana/dashboards
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: grafana-config
        configMap:
          name: grafana-config
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc
      - name: grafana-dashboards
        configMap:
          name: grafana-dashboards
```

---

## 🔹 Logging Architecture

### Logging Components

```yaml
# Logging Stack:
Log Collection:
  - Fluentd/Fluent Bit (log aggregation)
  - Filebeat (lightweight shipper)
  - Promtail (Loki log collector)

Log Storage:
  - Elasticsearch (search and analytics)
  - Loki (log aggregation system)
  - CloudWatch/Stackdriver

Log Visualization:
  - Kibana (Elasticsearch UI)
  - Grafana (Loki UI)
  - Cloud provider dashboards
```

### Fluentd DaemonSet

```yaml
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
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.15-debian-elasticsearch7-1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        - name: FLUENTD_SYSTEMD_CONF
          value: disable
        - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
          value: /var/log/containers/fluent*
        - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
          value: /^(?<time>.+) (?<stream>stdout|stderr)( (?<logtag>.))? (?<log>.*)$/
        resources:
          limits:
            memory: 512Mi
            cpu: 100m
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc
        livenessProbe:
          tcpSocket:
            port: 24224
          initialDelaySeconds: 30
          timeoutSeconds: 30
        readinessProbe:
          tcpSocket:
            port: 24224
          initialDelaySeconds: 5
          timeoutSeconds: 10
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluentd-config
        configMap:
          name: fluentd-config
```

### Fluentd Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    @include systemd.conf
    @include kubernetes.conf
    
    <match **>
      @type elasticsearch
      @id out_es
      @log_level info
      include_tag_key true
      host "#{ENV['FLUENT_ELASTICSEARCH_HOST']}"
      port "#{ENV['FLUENT_ELASTICSEARCH_PORT']}"
      scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'http'}"
      ssl_verify "#{ENV['FLUENT_ELASTICSEARCH_SSL_VERIFY'] || 'true'}"
      ssl_version "#{ENV['FLUENT_ELASTICSEARCH_SSL_VERSION'] || 'TLSv1_2'}"
      reload_connections "#{ENV['FLUENT_ELASTICSEARCH_RELOAD_CONNECTIONS'] || 'false'}"
      reconnect_on_error "#{ENV['FLUENT_ELASTICSEARCH_RECONNECT_ON_ERROR'] || 'true'}"
      reload_on_failure "#{ENV['FLUENT_ELASTICSEARCH_RELOAD_ON_FAILURE'] || 'true'}"
      log_es_400_reason "#{ENV['FLUENT_ELASTICSEARCH_LOG_ES_400_REASON'] || 'false'}"
      logstash_prefix "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_PREFIX'] || 'logstash'}"
      logstash_format true
      index_name "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_INDEX_NAME'] || 'logstash'}"
      type_name "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_TYPE_NAME'] || 'fluentd'}"
      <buffer>
        flush_thread_count "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_FLUSH_THREAD_COUNT'] || '8'}"
        flush_interval "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_FLUSH_INTERVAL'] || '5s'}"
        chunk_limit_size "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_CHUNK_LIMIT_SIZE'] || '2M'}"
        queue_limit_length "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_QUEUE_LIMIT_LENGTH'] || '32'}"
        retry_max_interval "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_RETRY_MAX_INTERVAL'] || '30'}"
        retry_forever true
      </buffer>
    </match>
  
  kubernetes.conf: |
    <match fluent.**>
      @type null
    </match>
    
    <source>
      @type tail
      @id in_tail_container_logs
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag raw.kubernetes.*
      read_from_head true
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key time
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>
    
    <match raw.kubernetes.**>
      @id raw.kubernetes
      @type detect_exceptions
      remove_tag_prefix raw
      message log
      stream stream
      multiline_flush_interval 5
      max_bytes 500000
      max_lines 1000
    </match>
    
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "#{ENV['FLUENT_FILTER_KUBERNETES_URL'] || 'https://' + ENV.fetch('KUBERNETES_SERVICE_HOST') + ':' + ENV.fetch('KUBERNETES_SERVICE_PORT') + '/api'}"
      verify_ssl "#{ENV['KUBERNETES_VERIFY_SSL'] || true}"
      ca_file "#{ENV['KUBERNETES_CA_FILE']}"
      skip_labels "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_LABELS'] || 'false'}"
      skip_container_metadata "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_CONTAINER_METADATA'] || 'false'}"
      skip_master_url "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_MASTER_URL'] || 'false'}"
      skip_namespace_metadata "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_NAMESPACE_METADATA'] || 'false'}"
    </filter>
```

---

## 🔹 ELK/EFK Stack

### Elasticsearch Deployment

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        env:
        - name: cluster.name
          value: k8s-logs
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 100m
            memory: 1Gi
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        livenessProbe:
          httpGet:
            scheme: HTTP
            path: /_cluster/health?local=true
            port: 9200
          initialDelaySeconds: 30
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            scheme: HTTP
            path: /_cluster/health?local=true
            port: 9200
          initialDelaySeconds: 5
          timeoutSeconds: 10
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

### Kibana Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.0
        ports:
        - containerPort: 5601
        env:
        - name: ELASTICSEARCH_URL
          value: http://elasticsearch:9200
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch:9200
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 30
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 30
          timeoutSeconds: 30
```

---

## 🔹 Health Checks & Probes

### Application with All Probe Types

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-with-probes
spec:
  replicas: 3
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
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: http
        - containerPort: 9090
          name: metrics
        
        # Startup probe for slow-starting applications
        startupProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30  # 5 minutes max startup time
          successThreshold: 1
        
        # Liveness probe to restart unhealthy containers
        livenessProbe:
          httpGet:
            path: /health
            port: 80
            httpHeaders:
            - name: Custom-Header
              value: liveness-check
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        
        # Readiness probe to control traffic routing
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 1
        
        # Resource limits for proper scheduling
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # Environment variables for health endpoints
        env:
        - name: HEALTH_CHECK_ENABLED
          value: "true"
        - name: METRICS_ENABLED
          value: "true"
```

### Custom Health Check Endpoints

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: health-check-config
data:
  health.sh: |
    #!/bin/bash
    # Custom health check script
    
    # Check database connectivity
    if ! nc -z database 5432; then
        echo "Database connection failed"
        exit 1
    fi
    
    # Check Redis connectivity
    if ! nc -z redis 6379; then
        echo "Redis connection failed"
        exit 1
    fi
    
    # Check application-specific health
    if ! curl -f http://localhost:8080/internal/health; then
        echo "Application health check failed"
        exit 1
    fi
    
    echo "All health checks passed"
    exit 0
  
  ready.sh: |
    #!/bin/bash
    # Custom readiness check script
    
    # Check if application is ready to serve traffic
    if ! curl -f http://localhost:8080/internal/ready; then
        echo "Application not ready"
        exit 1
    fi
    
    # Check if all dependencies are available
    if ! curl -f http://localhost:8080/internal/dependencies; then
        echo "Dependencies not ready"
        exit 1
    fi
    
    echo "Application ready"
    exit 0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-custom-probes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: custom-app
  template:
    metadata:
      labels:
        app: custom-app
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        ports:
        - containerPort: 8080
        
        # Custom exec probes
        livenessProbe:
          exec:
            command:
            - /bin/bash
            - /scripts/health.sh
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - /scripts/ready.sh
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 5
          failureThreshold: 3
        
        # TCP socket probe for non-HTTP services
        # livenessProbe:
        #   tcpSocket:
        #     port: 8080
        #   initialDelaySeconds: 30
        #   periodSeconds: 10
        
        volumeMounts:
        - name: health-scripts
          mountPath: /scripts
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
      
      volumes:
      - name: health-scripts
        configMap:
          name: health-check-config
          defaultMode: 0755
```

---

## 🔹 Metrics & Alerting

### ServiceMonitor for Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-app-metrics
  namespace: monitoring
  labels:
    app: web-app
spec:
  selector:
    matchLabels:
      app: web-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    honorLabels: true
  namespaceSelector:
    matchNames:
    - production
    - staging
```

### Prometheus Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alerts
  namespace: monitoring
spec:
  groups:
  - name: kubernetes.rules
    rules:
    # High CPU usage alert
    - alert: HighCPUUsage
      expr: (100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
        description: "CPU usage is above 80% for more than 5 minutes on {{ $labels.instance }}"
    
    # High memory usage alert
    - alert: HighMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage detected"
        description: "Memory usage is above 85% for more than 5 minutes on {{ $labels.instance }}"
    
    # Pod crash looping
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod is crash looping"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
    
    # Pod not ready
    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="false"} == 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod not ready"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been not ready for more than 10 minutes"
    
    # Deployment replica mismatch
    - alert: DeploymentReplicasMismatch
      expr: kube_deployment_spec_replicas != kube_deployment_status_available_replicas
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Deployment replicas mismatch"
        description: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has {{ $labels.spec_replicas }} desired but {{ $labels.available_replicas }} available replicas"
    
    # Persistent Volume usage
    - alert: PersistentVolumeUsageHigh
      expr: (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100 > 85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Persistent Volume usage high"
        description: "PV usage is above 85% for {{ $labels.persistentvolumeclaim }} in {{ $labels.namespace }}"
```

### Alertmanager Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'alerts@company.com'
      smtp_auth_username: 'alerts@company.com'
      smtp_auth_password: 'password'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'default'
      routes:
      - match:
          severity: critical
        receiver: 'critical-alerts'
        group_wait: 5s
        repeat_interval: 30m
      - match:
          severity: warning
        receiver: 'warning-alerts'
        repeat_interval: 2h
    
    receivers:
    - name: 'default'
      email_configs:
      - to: 'devops@company.com'
        subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Labels: {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
          {{ end }}
    
    - name: 'critical-alerts'
      slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts-critical'
        title: 'Critical Alert: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}
      pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_INTEGRATION_KEY'
        description: '{{ .GroupLabels.alertname }}: {{ .Alerts | len }} alerts'
    
    - name: 'warning-alerts'
      slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts-warning'
        title: 'Warning: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          {{ end }}
    
    inhibit_rules:
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname', 'cluster', 'service']
```

---

## 🔹 Cluster Maintenance

### Node Maintenance

```yaml
# Drain node for maintenance
apiVersion: v1
kind: Pod
metadata:
  name: node-maintenance
spec:
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
    env:
    - name: NODE_NAME
      value: "worker-node-1"
  restartPolicy: Never
---
# Commands for node maintenance:
# kubectl cordon $NODE_NAME              # Mark node as unschedulable
# kubectl drain $NODE_NAME --ignore-daemonsets --delete-emptydir-data
# # Perform maintenance
# kubectl uncordon $NODE_NAME            # Mark node as schedulable
```

### Cluster Upgrade Strategy

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: upgrade-strategy
data:
  upgrade-plan.md: |
    # Kubernetes Cluster Upgrade Plan
    
    ## Pre-upgrade Checklist:
    - [ ] Backup etcd
    - [ ] Backup application data
    - [ ] Test upgrade in staging
    - [ ] Check addon compatibility
    - [ ] Notify stakeholders
    
    ## Upgrade Process:
    1. Upgrade control plane nodes (one by one)
    2. Upgrade worker nodes (rolling update)
    3. Upgrade cluster addons
    4. Verify cluster functionality
    
    ## Post-upgrade Verification:
    - [ ] All nodes are Ready
    - [ ] All pods are Running
    - [ ] Services are accessible
    - [ ] Monitoring is functional
    - [ ] Applications are healthy
  
  upgrade-script.sh: |
    #!/bin/bash
    set -e
    
    # Upgrade control plane
    echo "Upgrading control plane..."
    kubeadm upgrade plan
    kubeadm upgrade apply v1.25.0
    
    # Upgrade kubelet and kubectl
    apt-mark unhold kubeadm kubelet kubectl
    apt-get update
    apt-get install -y kubeadm=1.25.0-00 kubelet=1.25.0-00 kubectl=1.25.0-00
    apt-mark hold kubeadm kubelet kubectl
    
    # Restart kubelet
    systemctl daemon-reload
    systemctl restart kubelet
    
    echo "Control plane upgrade completed"
```

### Backup and Recovery

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: k8s.gcr.io/etcd:3.5.0-0
            command:
            - /bin/sh
            - -c
            - |
              ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
              
              # Keep only last 7 days of backups
              find /backup -name "etcd-snapshot-*.db" -mtime +7 -delete
            
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup-storage
              mountPath: /backup
          
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup-storage
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
          
          restartPolicy: OnFailure
          hostNetwork: true
```

---

## 🔹 Performance Tuning

### Resource Optimization

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: optimized-app
  template:
    metadata:
      labels:
        app: optimized-app
    spec:
      # Node selection and affinity
      nodeSelector:
        node-type: compute-optimized
      
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
                  - optimized-app
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: app
        image: myapp:v1.0
        
        # Optimized resource allocation
        resources:
          requests:
            cpu: 100m      # Guaranteed CPU
            memory: 128Mi  # Guaranteed memory
          limits:
            cpu: 500m      # Maximum CPU
            memory: 512Mi  # Maximum memory
        
        # JVM tuning for Java applications
        env:
        - name: JAVA_OPTS
          value: "-Xms128m -Xmx512m -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
        
        # Optimized probes
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 30      # Less frequent checks
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10      # Moderate frequency
          timeoutSeconds: 3
          failureThreshold: 3
      
      # Optimize pod startup
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      dnsConfig:
        options:
        - name: ndots
          value: "2"
        - name: edns0
```

### HPA with Custom Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: optimized-app
  
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metric scaling (requests per second)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Max
```

---

## 🔹 Interview Questions

### Monitoring Questions

**Q: How do you monitor a Kubernetes cluster?**
A: Multi-layer monitoring approach:
- **Infrastructure**: Node Exporter for node metrics, cAdvisor for container metrics
- **Kubernetes**: kube-state-metrics for K8s object metrics
- **Applications**: Custom metrics via Prometheus client libraries
- **Logs**: Centralized logging with ELK/EFK stack
- **Visualization**: Grafana dashboards for metrics, Kibana for logs

**Q: Explain the difference between liveness and readiness probes.**
A: 
- **Liveness Probe**: Determines if container should be restarted (health check)
- **Readiness Probe**: Determines if container should receive traffic (traffic routing)
- **Startup Probe**: Gives slow-starting containers time to initialize before other probes

**Q: How do you handle log aggregation in Kubernetes?**
A: Using log aggregation stack:
- **Collection**: Fluentd/Fluent Bit DaemonSet on each node
- **Storage**: Elasticsearch cluster for searchable storage
- **Visualization**: Kibana for log analysis and dashboards
- **Retention**: Automated log rotation and cleanup policies

### Maintenance Questions

**Q: How do you perform cluster upgrades?**
A: Rolling upgrade strategy:
1. **Backup**: etcd and application data
2. **Control Plane**: Upgrade master nodes one by one
3. **Worker Nodes**: Drain, upgrade, uncordon nodes
4. **Addons**: Update cluster components
5. **Verification**: Test all functionality

**Q: How do you handle node maintenance?**
A: Node maintenance process:
1. **Cordon**: Mark node unschedulable
2. **Drain**: Evict pods gracefully
3. **Maintain**: Perform required maintenance
4. **Uncordon**: Mark node schedulable again
5. **Verify**: Ensure node is healthy

**Q: What's your backup strategy for Kubernetes?**
A: Comprehensive backup approach:
- **etcd**: Regular snapshots of cluster state
- **Persistent Volumes**: Application data backups
- **Configuration**: GitOps for declarative configs
- **Secrets**: Secure backup of sensitive data
- **Testing**: Regular restore testing

This comprehensive guide covers all essential monitoring, logging, and maintenance aspects for senior DevOps engineer interviews, including practical examples, troubleshooting scenarios, and operational best practices.
# 📊 Google Cloud Operations Complete Guide
*DevOps-Focused Reference*

## Table of Contents
1. [Cloud Operations Overview](#cloud-operations-overview)
2. [Cloud Monitoring](#cloud-monitoring)
3. [Cloud Logging](#cloud-logging)
4. [Cloud Trace](#cloud-trace)
5. [Cloud Debugger](#cloud-debugger)
6. [Cloud Profiler](#cloud-profiler)
7. [Error Reporting](#error-reporting)
8. [DevOps Integration](#devops-integration)
9. [Best Practices](#best-practices)

---

## 🔹 Cloud Operations Overview

### Key Questions for Cloud Operations

**Application Health Monitoring:**
- Is my application healthy and responding correctly?
- Are users experiencing any performance issues?
- What is the current error rate and response time?

**Infrastructure Monitoring:**
- Does my database have enough storage space?
- Are my servers running at optimum capacity?
- Is my network performing well?

**Business Impact:**
- Are SLAs being met?
- What is the user experience quality?
- How do incidents affect business metrics?

### Cloud Operations Suite Components

```yaml
Cloud Operations Suite (formerly Stackdriver):
  Monitoring: Infrastructure and application metrics
  Logging: Centralized log management and analysis
  Trace: Distributed tracing for microservices
  Debugger: Production debugging without code changes
  Profiler: Performance profiling and optimization
  Error Reporting: Real-time error tracking and alerting
```

---

## 🔹 Cloud Monitoring

### Core Concepts

**Metrics Collection:**
- **System metrics** - CPU, memory, disk, network
- **Application metrics** - Custom business metrics
- **Service metrics** - API response times, error rates
- **Infrastructure metrics** - Load balancer, database performance

### Workspace Configuration

**Creating and Managing Workspaces:**
```bash
# Create workspace using gcloud
gcloud alpha monitoring workspaces create \
    --display-name="Production Monitoring" \
    --description="Workspace for production environment"

# Add projects to workspace
gcloud alpha monitoring workspaces add-project \
    --workspace-project=monitoring-project \
    --monitored-project=app-project-1

# List workspaces
gcloud alpha monitoring workspaces list
```

**Workspace Best Practices:**
```yaml
Workspace Organization:
  Host Project: Dedicated monitoring project
  Monitored Projects: 
    - Production applications
    - Staging environments
    - Development projects
  
Multi-Cloud Monitoring:
  - GCP projects
  - AWS accounts (via AWS connector)
  - On-premises resources
  
Access Control:
  - Monitoring Viewer: Read-only access
  - Monitoring Editor: Create/modify dashboards
  - Monitoring Admin: Full workspace management
```

### Virtual Machine Monitoring

**Default Metrics:**
```yaml
Automatic VM Metrics:
  CPU:
    - compute.googleapis.com/instance/cpu/utilization
    - compute.googleapis.com/instance/cpu/usage_time
  
  Disk:
    - compute.googleapis.com/instance/disk/read_bytes_count
    - compute.googleapis.com/instance/disk/write_bytes_count
    - compute.googleapis.com/instance/disk/read_ops_count
    - compute.googleapis.com/instance/disk/write_ops_count
  
  Network:
    - compute.googleapis.com/instance/network/received_bytes_count
    - compute.googleapis.com/instance/network/sent_bytes_count
  
  Uptime:
    - compute.googleapis.com/instance/up
```

**Enhanced Monitoring with Ops Agent:**
```bash
# Install Ops Agent (replaces legacy agents)
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# Configure Ops Agent
sudo tee /etc/google-cloud-ops-agent/config.yaml << EOF
logging:
  receivers:
    syslog:
      type: files
      include_paths:
        - /var/log/syslog
        - /var/log/messages
    apache_access:
      type: apache_access
      include_paths:
        - /var/log/apache2/access.log
    apache_error:
      type: apache_error
      include_paths:
        - /var/log/apache2/error.log
  
  processors:
    batch:
      type: batch
  
  exporters:
    google:
      type: google_cloud_logging
  
  service:
    pipelines:
      default_pipeline:
        receivers: [syslog, apache_access, apache_error]
        processors: [batch]
        exporters: [google]

metrics:
  receivers:
    hostmetrics:
      type: hostmetrics
      collection_interval: 60s
    apache:
      type: apache
      endpoint: http://localhost/server-status?auto
  
  processors:
    batch:
      type: batch
  
  exporters:
    google:
      type: google_cloud_monitoring
  
  service:
    pipelines:
      default_pipeline:
        receivers: [hostmetrics, apache]
        processors: [batch]
        exporters: [google]
EOF

# Restart Ops Agent
sudo systemctl restart google-cloud-ops-agent
```

### Custom Metrics and Dashboards

**Creating Custom Metrics:**
```python
#!/usr/bin/env python3
# custom-metrics.py

from google.cloud import monitoring_v3
import time
import psutil

def create_custom_metric():
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{PROJECT_ID}"
    
    descriptor = monitoring_v3.MetricDescriptor()
    descriptor.type = "custom.googleapis.com/application/active_users"
    descriptor.metric_kind = monitoring_v3.MetricDescriptor.MetricKind.GAUGE
    descriptor.value_type = monitoring_v3.MetricDescriptor.ValueType.INT64
    descriptor.description = "Number of active users in the application"
    
    descriptor = client.create_metric_descriptor(
        name=project_name, metric_descriptor=descriptor
    )
    
    print(f"Created metric descriptor: {descriptor.name}")

def write_custom_metric(value):
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{PROJECT_ID}"
    
    series = monitoring_v3.TimeSeries()
    series.metric.type = "custom.googleapis.com/application/active_users"
    series.resource.type = "gce_instance"
    series.resource.labels["instance_id"] = INSTANCE_ID
    series.resource.labels["zone"] = ZONE
    
    now = time.time()
    seconds = int(now)
    nanos = int((now - seconds) * 10 ** 9)
    interval = monitoring_v3.TimeInterval(
        {"end_time": {"seconds": seconds, "nanos": nanos}}
    )
    point = monitoring_v3.Point(
        {"interval": interval, "value": {"int64_value": value}}
    )
    series.points = [point]
    
    client.create_time_series(name=project_name, time_series=[series])

def monitor_system_metrics():
    """Send system metrics to Cloud Monitoring"""
    
    while True:
        # CPU usage
        cpu_percent = psutil.cpu_percent(interval=1)
        write_custom_metric_value("cpu_usage_percent", cpu_percent)
        
        # Memory usage
        memory = psutil.virtual_memory()
        write_custom_metric_value("memory_usage_percent", memory.percent)
        
        # Disk usage
        disk = psutil.disk_usage('/')
        disk_percent = (disk.used / disk.total) * 100
        write_custom_metric_value("disk_usage_percent", disk_percent)
        
        time.sleep(60)  # Send metrics every minute

if __name__ == "__main__":
    PROJECT_ID = "my-project"
    INSTANCE_ID = "my-instance"
    ZONE = "us-central1-a"
    
    create_custom_metric()
    monitor_system_metrics()
```

**Dashboard Configuration:**
```json
{
  "displayName": "Application Performance Dashboard",
  "mosaicLayout": {
    "tiles": [
      {
        "width": 6,
        "height": 4,
        "widget": {
          "title": "CPU Utilization",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\" resource.type=\"gce_instance\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_MEAN",
                      "crossSeriesReducer": "REDUCE_MEAN",
                      "groupByFields": ["resource.label.instance_name"]
                    }
                  }
                }
              }
            ],
            "timeshiftDuration": "0s",
            "yAxis": {
              "label": "CPU Utilization",
              "scale": "LINEAR"
            }
          }
        }
      },
      {
        "width": 6,
        "height": 4,
        "xPos": 6,
        "widget": {
          "title": "Application Response Time",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "metric.type=\"custom.googleapis.com/application/response_time\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_MEAN"
                    }
                  }
                }
              }
            ]
          }
        }
      }
    ]
  }
}
```

### Alerting Policies

**Comprehensive Alerting Setup:**
```yaml
# alerting-policy.yaml
displayName: "Production Application Alerts"
conditions:
  - displayName: "High CPU Usage"
    conditionThreshold:
      filter: 'resource.type="gce_instance" AND metric.type="compute.googleapis.com/instance/cpu/utilization"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 0.8
      duration: 300s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_MEAN
          crossSeriesReducer: REDUCE_MEAN
          groupByFields:
            - resource.label.instance_name

  - displayName: "High Error Rate"
    conditionThreshold:
      filter: 'resource.type="cloud_run_revision" AND metric.type="run.googleapis.com/request_count"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 100
      duration: 180s

  - displayName: "Database Connection Pool Exhaustion"
    conditionThreshold:
      filter: 'resource.type="cloudsql_database" AND metric.type="cloudsql.googleapis.com/database/postgresql/num_backends"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 80
      duration: 120s

notificationChannels:
  - type: "slack"
    labels:
      channel_name: "#alerts"
      url: "https://hooks.slack.com/services/..."
  
  - type: "email"
    labels:
      email_address: "devops@company.com"
  
  - type: "sms"
    labels:
      number: "+1234567890"

documentation:
  content: |
    ## Production Alert Runbook
    
    ### High CPU Usage
    1. Check application logs for errors
    2. Verify if auto-scaling is working
    3. Consider scaling up if sustained high usage
    
    ### High Error Rate
    1. Check application error logs
    2. Verify database connectivity
    3. Check external service dependencies
    
    ### Database Issues
    1. Check connection pool settings
    2. Verify database performance metrics
    3. Consider read replicas for read-heavy workloads
  
  mimeType: "text/markdown"
```

**Creating Alerts with gcloud:**
```bash
# Create alerting policy
gcloud alpha monitoring policies create --policy-from-file=alerting-policy.yaml

# Create notification channels
gcloud alpha monitoring channels create \
    --display-name="DevOps Slack" \
    --type=slack \
    --channel-labels=channel_name="#alerts",url="https://hooks.slack.com/..."

gcloud alpha monitoring channels create \
    --display-name="DevOps Email" \
    --type=email \
    --channel-labels=email_address="devops@company.com"

# List existing policies
gcloud alpha monitoring policies list

# Update policy
gcloud alpha monitoring policies update POLICY_ID --policy-from-file=updated-policy.yaml
```

---

## 🔹 Cloud Logging

### Log Collection Architecture

**Automatic Log Collection:**
```yaml
GCP Services with Automatic Logging:
  Compute:
    - App Engine: Application and request logs
    - Cloud Run: Container stdout/stderr
    - Cloud Functions: Function execution logs
    - GKE: Container and cluster logs
  
  Data Services:
    - Cloud SQL: Error and slow query logs
    - Cloud Storage: Access logs (when enabled)
    - BigQuery: Job and audit logs
  
  Networking:
    - Load Balancer: Access logs
    - VPC Flow Logs: Network traffic logs
    - Cloud CDN: Cache hit/miss logs
```

**Manual Log Collection Setup:**
```bash
# Install Ops Agent on Compute Engine
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# Configure application log collection
sudo tee /etc/google-cloud-ops-agent/config.yaml << EOF
logging:
  receivers:
    app_logs:
      type: files
      include_paths:
        - /var/log/myapp/*.log
      exclude_paths:
        - /var/log/myapp/debug.log
    
    nginx_access:
      type: nginx_access
      include_paths:
        - /var/log/nginx/access.log
    
    nginx_error:
      type: nginx_error
      include_paths:
        - /var/log/nginx/error.log
    
    syslog:
      type: syslog
      listen_host: 0.0.0.0
      listen_port: 514
      protocol: udp
  
  processors:
    parse_json:
      type: parse_json
      field: message
    
    add_metadata:
      type: modify_fields
      fields:
        environment:
          static_value: "production"
        service:
          static_value: "web-app"
  
  exporters:
    google_cloud:
      type: google_cloud_logging
  
  service:
    pipelines:
      default:
        receivers: [app_logs, nginx_access, nginx_error, syslog]
        processors: [parse_json, add_metadata]
        exporters: [google_cloud]
EOF

sudo systemctl restart google-cloud-ops-agent
```

### Audit and Security Logs

**Cloud Audit Logs Configuration:**
```yaml
# audit-policy.yaml
auditConfigs:
- service: allServices
  auditLogConfigs:
  - logType: ADMIN_READ
  - logType: DATA_READ
  - logType: DATA_WRITE

- service: storage.googleapis.com
  auditLogConfigs:
  - logType: DATA_READ
  - logType: DATA_WRITE
    exemptedMembers:
    - serviceAccount:backup-service@project.iam.gserviceaccount.com

- service: compute.googleapis.com
  auditLogConfigs:
  - logType: ADMIN_READ
  - logType: DATA_WRITE
```

**Audit Log Analysis:**
```bash
# Query admin activity logs
gcloud logging read '
  protoPayload.serviceName="compute.googleapis.com" AND
  protoPayload.methodName="v1.compute.instances.insert" AND
  timestamp>="2024-01-01T00:00:00Z"
' --limit=50 --format=json

# Query data access logs
gcloud logging read '
  protoPayload.serviceName="storage.googleapis.com" AND
  protoPayload.methodName="storage.objects.get" AND
  protoPayload.authenticationInfo.principalEmail="user@company.com"
' --limit=100

# Query policy denied logs
gcloud logging read '
  protoPayload.serviceName="cloudresourcemanager.googleapis.com" AND
  severity="ERROR" AND
  protoPayload.status.code=7
' --limit=20
```

### Log Routing and Export

**Log Router Configuration:**
```bash
# Create log sink to Cloud Storage
gcloud logging sinks create audit-logs-storage \
    storage.googleapis.com/audit-logs-bucket \
    --log-filter='protoPayload.serviceName="compute.googleapis.com" OR protoPayload.serviceName="storage.googleapis.com"'

# Create log sink to BigQuery
gcloud logging sinks create app-logs-bigquery \
    bigquery.googleapis.com/projects/my-project/datasets/application_logs \
    --log-filter='resource.type="gce_instance" AND labels.application="web-app"'

# Create log sink to Pub/Sub
gcloud logging sinks create realtime-alerts \
    pubsub.googleapis.com/projects/my-project/topics/log-alerts \
    --log-filter='severity>=ERROR AND resource.type="cloud_run_revision"'

# Exclude debug logs from default sink
gcloud logging sinks update _Default \
    --log-filter='NOT (severity="DEBUG" OR labels.log_level="debug")'
```

**Advanced Log Processing:**
```python
#!/usr/bin/env python3
# log-processor.py

from google.cloud import logging
from google.cloud import pubsub_v1
import json
import re

class LogProcessor:
    def __init__(self, project_id):
        self.logging_client = logging.Client(project=project_id)
        self.publisher = pubsub_v1.PublisherClient()
        self.project_id = project_id
    
    def process_error_logs(self):
        """Process error logs and send alerts"""
        
        filter_str = '''
            severity>=ERROR AND
            timestamp>="2024-01-01T00:00:00Z" AND
            resource.type="cloud_run_revision"
        '''
        
        entries = self.logging_client.list_entries(filter_=filter_str)
        
        for entry in entries:
            if self.is_critical_error(entry):
                self.send_alert(entry)
    
    def is_critical_error(self, entry):
        """Determine if log entry represents critical error"""
        
        critical_patterns = [
            r'OutOfMemoryError',
            r'DatabaseConnectionError',
            r'ServiceUnavailable',
            r'TimeoutException'
        ]
        
        log_text = str(entry.payload)
        
        for pattern in critical_patterns:
            if re.search(pattern, log_text, re.IGNORECASE):
                return True
        
        return False
    
    def send_alert(self, entry):
        """Send alert to Pub/Sub topic"""
        
        alert_data = {
            'timestamp': entry.timestamp.isoformat(),
            'severity': entry.severity,
            'resource': dict(entry.resource.labels),
            'message': str(entry.payload),
            'log_name': entry.log_name
        }
        
        topic_path = self.publisher.topic_path(self.project_id, 'critical-alerts')
        
        self.publisher.publish(
            topic_path,
            json.dumps(alert_data).encode('utf-8')
        )
    
    def create_log_metrics(self):
        """Create log-based metrics"""
        
        # Error rate metric
        error_metric = {
            'name': f'projects/{self.project_id}/metrics/error_rate',
            'description': 'Rate of error logs',
            'filter': 'severity>=ERROR',
            'metricDescriptor': {
                'metricKind': 'GAUGE',
                'valueType': 'INT64'
            }
        }
        
        # Response time metric from logs
        response_time_metric = {
            'name': f'projects/{self.project_id}/metrics/response_time',
            'description': 'Application response time from logs',
            'filter': 'jsonPayload.response_time_ms>0',
            'valueExtractor': 'EXTRACT(jsonPayload.response_time_ms)',
            'metricDescriptor': {
                'metricKind': 'GAUGE',
                'valueType': 'DOUBLE'
            }
        }

if __name__ == "__main__":
    processor = LogProcessor("my-project")
    processor.process_error_logs()
    processor.create_log_metrics()
```

### Log Analysis and Querying

**Advanced Log Queries:**
```bash
# Find all failed authentication attempts
gcloud logging read '
  protoPayload.methodName="google.iam.admin.v1.CreateServiceAccountKey" AND
  protoPayload.status.code!=0
' --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.status.message)"

# Analyze application performance
gcloud logging read '
  resource.type="cloud_run_revision" AND
  httpRequest.status>=400 AND
  timestamp>="2024-01-01T00:00:00Z"
' --format="table(timestamp,httpRequest.requestMethod,httpRequest.requestUrl,httpRequest.status,httpRequest.latency)"

# Monitor database queries
gcloud logging read '
  resource.type="cloudsql_database" AND
  jsonPayload.message~"slow query" AND
  timestamp>="2024-01-01T00:00:00Z"
' --format="value(jsonPayload.message)"

# Track resource usage patterns
gcloud logging read '
  resource.type="gce_instance" AND
  jsonPayload.cpu_usage>80 AND
  timestamp>="2024-01-01T00:00:00Z"
' --format="table(timestamp,resource.labels.instance_name,jsonPayload.cpu_usage,jsonPayload.memory_usage)"
```

---

## 🔹 Cloud Trace

### Distributed Tracing Setup

**Automatic Tracing for GCP Services:**
```yaml
Services with Built-in Tracing:
  - App Engine Standard and Flexible
  - Cloud Run (with trace headers)
  - Cloud Functions (automatic)
  - GKE (with Istio service mesh)
  - Compute Engine (with trace libraries)
```

**Manual Instrumentation:**
```python
#!/usr/bin/env python3
# trace-instrumentation.py

from google.cloud import trace_v1
from opencensus.ext.gcp import trace_exporter
from opencensus.trace import tracer as tracer_module
from opencensus.trace.samplers import ProbabilitySampler
import time
import requests

# Initialize tracer
exporter = trace_exporter.TraceExporter(project_id="my-project")
tracer = tracer_module.Tracer(
    exporter=exporter,
    sampler=ProbabilitySampler(rate=1.0)  # Sample 100% for development
)

def trace_database_operation():
    """Example of tracing database operations"""
    
    with tracer.span(name='database_query') as span:
        span.add_annotation('Starting database query')
        span.add_attribute('query_type', 'SELECT')
        span.add_attribute('table_name', 'users')
        
        # Simulate database query
        time.sleep(0.1)
        
        span.add_annotation('Database query completed')
        return {'user_count': 1000}

def trace_external_api_call():
    """Example of tracing external API calls"""
    
    with tracer.span(name='external_api_call') as span:
        span.add_attribute('api_endpoint', 'https://api.example.com/data')
        span.add_attribute('method', 'GET')
        
        try:
            response = requests.get('https://api.example.com/data', timeout=5)
            span.add_attribute('status_code', response.status_code)
            span.add_attribute('response_size', len(response.content))
            
            if response.status_code >= 400:
                span.add_annotation('API call failed')
            
            return response.json()
            
        except requests.exceptions.Timeout:
            span.add_annotation('API call timed out')
            raise
        except Exception as e:
            span.add_annotation(f'API call error: {str(e)}')
            raise

def main_application_flow():
    """Main application with distributed tracing"""
    
    with tracer.span(name='main_request') as span:
        span.add_attribute('user_id', '12345')
        span.add_attribute('request_type', 'user_profile')
        
        # Trace database operation
        user_data = trace_database_operation()
        
        # Trace external API call
        external_data = trace_external_api_call()
        
        # Process data
        with tracer.span(name='data_processing') as processing_span:
            processing_span.add_annotation('Processing user data')
            time.sleep(0.05)  # Simulate processing
            
            result = {
                'user_data': user_data,
                'external_data': external_data,
                'processed_at': time.time()
            }
        
        span.add_annotation('Request completed successfully')
        return result

if __name__ == "__main__":
    result = main_application_flow()
    print(f"Result: {result}")
```

**Microservices Tracing:**
```python
#!/usr/bin/env python3
# microservice-tracing.py

from flask import Flask, request
from opencensus.ext.flask.flask_middleware import FlaskMiddleware
from opencensus.ext.gcp import trace_exporter
from opencensus.trace.samplers import ProbabilitySampler
import requests

app = Flask(__name__)

# Configure tracing middleware
middleware = FlaskMiddleware(
    app,
    exporter=trace_exporter.TraceExporter(project_id="my-project"),
    sampler=ProbabilitySampler(rate=0.1)  # Sample 10% in production
)

@app.route('/api/user/<user_id>')
def get_user(user_id):
    """User service endpoint with tracing"""
    
    # Get current trace context
    tracer = middleware.tracer
    
    with tracer.span(name='get_user_profile') as span:
        span.add_attribute('user_id', user_id)
        
        # Call database service
        with tracer.span(name='database_lookup') as db_span:
            db_span.add_attribute('operation', 'SELECT')
            user_data = call_database_service(user_id)
        
        # Call external service with trace propagation
        with tracer.span(name='external_service_call') as ext_span:
            headers = {}
            tracer.add_attribute_to_current_span('service', 'user-preferences')
            
            # Propagate trace context
            trace_header = request.headers.get('X-Cloud-Trace-Context')
            if trace_header:
                headers['X-Cloud-Trace-Context'] = trace_header
            
            preferences = requests.get(
                f'http://preferences-service/api/preferences/{user_id}',
                headers=headers
            ).json()
        
        result = {
            'user': user_data,
            'preferences': preferences
        }
        
        span.add_attribute('response_size', len(str(result)))
        return result

def call_database_service(user_id):
    """Simulate database service call"""
    import time
    time.sleep(0.02)  # Simulate DB latency
    return {'id': user_id, 'name': 'John Doe', 'email': 'john@example.com'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080, debug=True)
```

### Trace Analysis

**Analyzing Performance Bottlenecks:**
```python
#!/usr/bin/env python3
# trace-analysis.py

from google.cloud import trace_v1
import statistics

class TraceAnalyzer:
    def __init__(self, project_id):
        self.client = trace_v1.TraceServiceClient()
        self.project_id = project_id
    
    def analyze_latency_patterns(self, hours=24):
        """Analyze latency patterns over time"""
        
        import datetime
        
        end_time = datetime.datetime.utcnow()
        start_time = end_time - datetime.timedelta(hours=hours)
        
        # Get traces
        traces = self.client.list_traces(
            project_id=self.project_id,
            start_time=start_time,
            end_time=end_time
        )
        
        latencies = []
        span_latencies = {}
        
        for trace in traces:
            for span in trace.spans:
                duration = (span.end_time - span.start_time).total_seconds() * 1000
                latencies.append(duration)
                
                span_name = span.name
                if span_name not in span_latencies:
                    span_latencies[span_name] = []
                span_latencies[span_name].append(duration)
        
        # Calculate statistics
        if latencies:
            analysis = {
                'total_traces': len(traces),
                'overall_stats': {
                    'mean': statistics.mean(latencies),
                    'median': statistics.median(latencies),
                    'p95': self.percentile(latencies, 95),
                    'p99': self.percentile(latencies, 99),
                    'max': max(latencies)
                },
                'span_analysis': {}
            }
            
            for span_name, durations in span_latencies.items():
                analysis['span_analysis'][span_name] = {
                    'count': len(durations),
                    'mean': statistics.mean(durations),
                    'p95': self.percentile(durations, 95),
                    'max': max(durations)
                }
            
            return analysis
        
        return None
    
    def percentile(self, data, percentile):
        """Calculate percentile"""
        sorted_data = sorted(data)
        index = int(len(sorted_data) * percentile / 100)
        return sorted_data[min(index, len(sorted_data) - 1)]
    
    def find_slow_operations(self, threshold_ms=1000):
        """Find operations slower than threshold"""
        
        # Implementation for finding slow operations
        pass

if __name__ == "__main__":
    analyzer = TraceAnalyzer("my-project")
    analysis = analyzer.analyze_latency_patterns(hours=24)
    
    if analysis:
        print(f"Analyzed {analysis['total_traces']} traces")
        print(f"Mean latency: {analysis['overall_stats']['mean']:.2f}ms")
        print(f"P95 latency: {analysis['overall_stats']['p95']:.2f}ms")
        print(f"P99 latency: {analysis['overall_stats']['p99']:.2f}ms")
```

---

## 🔹 Cloud Debugger

### Production Debugging Setup

**Debugger Agent Installation:**
```bash
# For Java applications
java -javaagent:cdbg_java_agent.jar \
     -Dcom.google.cdbg.module=my-app \
     -Dcom.google.cdbg.version=1.0 \
     -jar myapp.jar

# For Python applications
pip install google-python-cloud-debugger

# In Python code
try:
    import googleclouddebugger
    googleclouddebugger.enable(
        module='my-app',
        version='1.0'
    )
except ImportError:
    pass  # Debugger not available
```

**Debugger Configuration:**
```python
#!/usr/bin/env python3
# debugger-setup.py

import os
import googleclouddebugger

def setup_cloud_debugger():
    """Setup Cloud Debugger for production debugging"""
    
    # Only enable in production
    if os.environ.get('ENVIRONMENT') == 'production':
        try:
            googleclouddebugger.enable(
                module=os.environ.get('SERVICE_NAME', 'my-app'),
                version=os.environ.get('SERVICE_VERSION', '1.0'),
                # Optional: specify source context
                source_context={
                    'git': {
                        'revisionId': os.environ.get('GIT_COMMIT', 'unknown')
                    }
                }
            )
            print("Cloud Debugger enabled")
        except Exception as e:
            print(f"Failed to enable Cloud Debugger: {e}")

# Application code with debugger
def process_user_request(user_id, request_data):
    """Process user request with debugging capabilities"""
    
    # This function can be debugged in production
    user_profile = get_user_profile(user_id)
    
    if not user_profile:
        # Debugger can capture variables here
        error_context = {
            'user_id': user_id,
            'request_data': request_data,
            'timestamp': time.time()
        }
        raise ValueError(f"User not found: {user_id}")
    
    # Process request
    result = process_business_logic(user_profile, request_data)
    
    return result

if __name__ == "__main__":
    setup_cloud_debugger()
```

### Debugging Best Practices

**Snapshot and Logpoint Usage:**
```bash
# Create snapshot using gcloud
gcloud debug snapshots create \
    --target=my-app \
    --target-version=1.0 \
    --location=main.py:42 \
    --condition="user_id == '12345'"

# Create logpoint
gcloud debug logpoints create \
    --target=my-app \
    --target-version=1.0 \
    --location=main.py:55 \
    --format="User {user_id} processed with result {result}"

# List active snapshots
gcloud debug snapshots list --target=my-app

# Delete snapshot
gcloud debug snapshots delete SNAPSHOT_ID --target=my-app
```

---

## 🔹 Cloud Profiler

### Performance Profiling Setup

**Profiler Agent Integration:**
```python
#!/usr/bin/env python3
# profiler-setup.py

import googlecloudprofiler
import os

def setup_cloud_profiler():
    """Setup Cloud Profiler for performance analysis"""
    
    try:
        googlecloudprofiler.start(
            service=os.environ.get('SERVICE_NAME', 'my-app'),
            service_version=os.environ.get('SERVICE_VERSION', '1.0'),
            # Optional configuration
            verbose=3,  # Logging level
            disable_cpu_profiling=False,
            disable_heap_profiling=False,
            heap_sampling_interval=512 * 1024,  # 512KB
        )
        print("Cloud Profiler started successfully")
    except Exception as e:
        print(f"Failed to start Cloud Profiler: {e}")

# Example application with profiling
def cpu_intensive_function():
    """CPU-intensive function for profiling"""
    
    result = 0
    for i in range(1000000):
        result += i * i
    return result

def memory_intensive_function():
    """Memory-intensive function for profiling"""
    
    large_list = []
    for i in range(100000):
        large_list.append(f"Item {i}" * 10)
    return len(large_list)

if __name__ == "__main__":
    setup_cloud_profiler()
    
    # Run application code
    cpu_result = cpu_intensive_function()
    memory_result = memory_intensive_function()
```

**Profiler Analysis:**
```python
#!/usr/bin/env python3
# profiler-analysis.py

from google.cloud import profiler_v2
import time

class ProfilerAnalyzer:
    def __init__(self, project_id):
        self.client = profiler_v2.ProfilerServiceClient()
        self.project_id = project_id
    
    def analyze_cpu_usage(self, service_name, hours=24):
        """Analyze CPU usage patterns"""
        
        import datetime
        
        end_time = datetime.datetime.utcnow()
        start_time = end_time - datetime.timedelta(hours=hours)
        
        # Get CPU profiles
        profiles = self.client.list_profiles(
            parent=f"projects/{self.project_id}",
            filter=f'profile_type="CPU" AND target="{service_name}"',
            interval={
                'start_time': start_time,
                'end_time': end_time
            }
        )
        
        cpu_hotspots = []
        
        for profile in profiles:
            # Analyze profile data
            profile_data = self.client.get_profile(name=profile.name)
            
            # Extract hotspots (simplified)
            hotspots = self.extract_hotspots(profile_data)
            cpu_hotspots.extend(hotspots)
        
        return self.summarize_hotspots(cpu_hotspots)
    
    def extract_hotspots(self, profile_data):
        """Extract performance hotspots from profile"""
        # Implementation depends on profile format
        return []
    
    def summarize_hotspots(self, hotspots):
        """Summarize performance hotspots"""
        # Group and analyze hotspots
        return {
            'top_functions': [],
            'recommendations': []
        }

if __name__ == "__main__":
    analyzer = ProfilerAnalyzer("my-project")
    analysis = analyzer.analyze_cpu_usage("my-app", hours=24)
```

---

## 🔹 Error Reporting

### Error Collection and Analysis

**Automatic Error Reporting:**
```python
#!/usr/bin/env python3
# error-reporting.py

from google.cloud import error_reporting
import logging
import traceback

# Setup error reporting client
error_client = error_reporting.Client(project="my-project")

def setup_error_reporting():
    """Setup error reporting with logging integration"""
    
    # Configure logging to send errors to Error Reporting
    logging.basicConfig(level=logging.INFO)
    
    # Add error reporting handler
    handler = error_client.get_default_handler()
    logging.getLogger().addHandler(handler)

def report_error_with_context(error, user_id=None, request_id=None):
    """Report error with additional context"""
    
    try:
        # Report to Error Reporting with context
        error_client.report_exception(
            http_context={
                'method': 'POST',
                'url': '/api/process',
                'userAgent': 'MyApp/1.0',
                'remoteIp': '192.168.1.100'
            },
            user=user_id or 'anonymous'
        )
        
        # Also log with structured data
        logging.error(
            "Application error occurred",
            extra={
                'user_id': user_id,
                'request_id': request_id,
                'error_type': type(error).__name__,
                'error_message': str(error),
                'stack_trace': traceback.format_exc()
            }
        )
        
    except Exception as reporting_error:
        # Fallback logging if error reporting fails
        logging.error(f"Failed to report error: {reporting_error}")
        logging.error(f"Original error: {error}")

def application_with_error_handling():
    """Application code with comprehensive error handling"""
    
    try:
        # Simulate application logic
        result = risky_operation()
        return result
        
    except ValueError as e:
        # Handle specific error types
        report_error_with_context(e, user_id="12345", request_id="req-789")
        return {"error": "Invalid input provided"}
        
    except Exception as e:
        # Handle unexpected errors
        report_error_with_context(e, user_id="12345", request_id="req-789")
        return {"error": "Internal server error"}

def risky_operation():
    """Simulate operation that might fail"""
    import random
    
    if random.random() < 0.3:  # 30% chance of failure
        raise ValueError("Invalid data format")
    
    if random.random() < 0.1:  # 10% chance of unexpected error
        raise RuntimeError("Unexpected system error")
    
    return {"status": "success", "data": "processed"}

if __name__ == "__main__":
    setup_error_reporting()
    
    # Run application
    for i in range(10):
        result = application_with_error_handling()
        print(f"Request {i}: {result}")
```

**Error Analysis and Alerting:**
```python
#!/usr/bin/env python3
# error-analysis.py

from google.cloud import error_reporting
from google.cloud import monitoring_v3
import time

class ErrorAnalyzer:
    def __init__(self, project_id):
        self.error_client = error_reporting.Client(project=project_id)
        self.monitoring_client = monitoring_v3.MetricServiceClient()
        self.project_id = project_id
    
    def analyze_error_trends(self, hours=24):
        """Analyze error trends over time"""
        
        # Get error statistics
        stats = self.error_client.list_group_stats(
            project_name=f"projects/{self.project_id}",
            time_range={
                'period': f"{hours}h"
            }
        )
        
        error_analysis = {
            'total_errors': 0,
            'error_groups': [],
            'top_errors': [],
            'error_rate_trend': []
        }
        
        for stat in stats:
            error_analysis['total_errors'] += stat.count
            
            error_group = {
                'group_id': stat.group.group_id,
                'count': stat.count,
                'first_seen': stat.first_seen_time,
                'last_seen': stat.last_seen_time,
                'affected_users': stat.affected_users_count
            }
            
            error_analysis['error_groups'].append(error_group)
        
        # Sort by count to get top errors
        error_analysis['top_errors'] = sorted(
            error_analysis['error_groups'],
            key=lambda x: x['count'],
            reverse=True
        )[:10]
        
        return error_analysis
    
    def create_error_rate_alert(self, threshold=10):
        """Create alert for high error rates"""
        
        alert_policy = {
            'display_name': 'High Error Rate Alert',
            'conditions': [{
                'display_name': 'Error rate exceeds threshold',
                'condition_threshold': {
                    'filter': f'resource.type="global" AND metric.type="logging.googleapis.com/user/error_count"',
                    'comparison': 'COMPARISON_GREATER_THAN',
                    'threshold_value': threshold,
                    'duration': {'seconds': 300},  # 5 minutes
                    'aggregations': [{
                        'alignment_period': {'seconds': 60},
                        'per_series_aligner': 'ALIGN_RATE'
                    }]
                }
            }],
            'notification_channels': [
                # Add notification channel IDs
            ],
            'alert_strategy': {
                'auto_close': {'seconds': 86400}  # 24 hours
            }
        }
        
        # Create the alert policy
        project_name = f"projects/{self.project_id}"
        policy = self.monitoring_client.create_alert_policy(
            name=project_name,
            alert_policy=alert_policy
        )
        
        return policy

if __name__ == "__main__":
    analyzer = ErrorAnalyzer("my-project")
    
    # Analyze recent errors
    analysis = analyzer.analyze_error_trends(hours=24)
    print(f"Total errors in last 24h: {analysis['total_errors']}")
    
    # Show top errors
    for error in analysis['top_errors'][:5]:
        print(f"Error Group {error['group_id']}: {error['count']} occurrences")
    
    # Create error rate alert
    alert_policy = analyzer.create_error_rate_alert(threshold=50)
    print(f"Created alert policy: {alert_policy.name}")
```

This comprehensive Cloud Operations guide provides you with the knowledge and practical examples needed to master Google Cloud Operations from a DevOps perspective. The guide covers everything from basic monitoring setup to advanced error analysis and performance optimization techniques.
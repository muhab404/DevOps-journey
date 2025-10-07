# ⚡ Google Cloud Functions Study Guide
*DevOps-Focused Complete Reference*

## Table of Contents
1. [Cloud Functions Overview](#cloud-functions-overview)
2. [Event Triggers](#event-triggers)
3. [Product Versions (1st gen vs 2nd gen)](#product-versions)
4. [Function Examples](#function-examples)
5. [Scaling & Concurrency](#scaling--concurrency)
6. [Deployment with gcloud](#deployment-with-gcloud)
7. [Best Practices](#best-practices)
8. [Architecture Patterns](#architecture-patterns)
9. [DevOps Integration](#devops-integration)
10. [Monitoring & Troubleshooting](#monitoring--troubleshooting)

---

## 🔹 Cloud Functions Overview

### What is Cloud Functions?

**Cloud Functions** is a serverless execution environment for building and connecting cloud services. It's an event-driven compute service that runs your code in response to events without requiring server management.

**Core Characteristics:**
- **Event-driven** - Responds to cloud events automatically
- **Serverless compute** - No infrastructure management
- **Automatic scaling** - Scales from zero to thousands of instances
- **Highly available** - Built-in redundancy and fault tolerance
- **Pay-per-use** - Only pay for execution time and resources

### Pricing Model

**Billing Components:**
1. **Invocations** - Number of function executions
2. **Execution time** - CPU time during function execution
3. **Memory allocation** - RAM allocated to function
4. **Networking** - Egress traffic costs

**Cost Calculation:**
```python
def calculate_functions_cost(invocations_per_month, avg_duration_ms, memory_mb):
    # Invocation cost: $0.40 per million invocations
    invocation_cost = (invocations_per_month / 1000000) * 0.40
    
    # Compute cost (GB-seconds)
    gb_seconds = (invocations_per_month * avg_duration_ms / 1000) * (memory_mb / 1024)
    compute_cost = gb_seconds * 0.0000025  # $0.0000025 per GB-second
    
    # CPU cost (GHz-seconds) 
    ghz_seconds = (invocations_per_month * avg_duration_ms / 1000) * 0.4  # 400MHz default
    cpu_cost = ghz_seconds * 0.0000100  # $0.0000100 per GHz-second
    
    return {
        'invocation_cost': invocation_cost,
        'compute_cost': compute_cost,
        'cpu_cost': cpu_cost,
        'total_cost': invocation_cost + compute_cost + cpu_cost
    }

# Example: 1M invocations/month, 200ms avg, 256MB
cost = calculate_functions_cost(1000000, 200, 256)
print(f"Monthly cost: ${cost['total_cost']:.2f}")
```

### Supported Runtimes

**Available Runtimes:**
- **Node.js** - 16, 18, 20
- **Python** - 3.8, 3.9, 3.10, 3.11
- **Go** - 1.18, 1.19, 1.20, 1.21
- **Java** - 11, 17
- **.NET** - 6
- **Ruby** - 3.0, 3.1, 3.2

**Runtime Selection Criteria:**
```bash
# Performance comparison (cold start times)
Node.js:  ~100-300ms
Python:   ~200-500ms  
Go:       ~100-200ms
Java:     ~1-3s
.NET:     ~1-2s
Ruby:     ~300-800ms
```

---

## 🔹 Event Triggers

### Trigger Types Overview

**HTTP Triggers:**
- Direct HTTP requests (GET, POST, PUT, DELETE, OPTIONS)
- Webhook endpoints
- API integrations
- Web application backends

**Cloud Storage Triggers:**
- File upload (`google.storage.object.finalize`)
- File deletion (`google.storage.object.delete`)
- Metadata updates (`google.storage.object.metadataUpdate`)

**Pub/Sub Triggers:**
- Message publishing to topics
- Asynchronous processing
- Event-driven architectures
- Decoupled system communication

**Firebase Triggers:**
- Authentication events (user creation, deletion)
- Realtime Database changes
- Firestore document changes
- Remote Config updates

**Cloud Firestore Triggers:**
- Document creation (`providers/cloud.firestore/eventTypes/document.create`)
- Document updates (`providers/cloud.firestore/eventTypes/document.update`)
- Document deletion (`providers/cloud.firestore/eventTypes/document.delete`)

**Cloud Logging Triggers:**
- Log entry matching filters
- Error monitoring and alerting
- Audit trail processing

**Eventarc Triggers (2nd gen only):**
- 90+ event sources
- Cloud Audit Logs
- Custom events
- Third-party integrations

### Event Trigger Examples

**Cloud Storage Trigger:**
```python
def process_image(cloud_event):
    """Triggered by Cloud Storage object creation."""
    import functions_framework
    from google.cloud import vision
    
    # Get file info from event
    bucket = cloud_event.data['bucket']
    name = cloud_event.data['name']
    
    if not name.lower().endswith(('.jpg', '.jpeg', '.png')):
        return
    
    # Process image with Vision API
    client = vision.ImageAnnotatorClient()
    image = vision.Image()
    image.source.image_uri = f"gs://{bucket}/{name}"
    
    response = client.text_detection(image=image)
    texts = response.text_annotations
    
    # Store results
    if texts:
        print(f"Detected text in {name}: {texts[0].description}")
```

**Pub/Sub Trigger:**
```python
import base64
import json

def process_pubsub_message(cloud_event):
    """Triggered by Pub/Sub message."""
    # Decode message
    pubsub_message = base64.b64decode(cloud_event.data['message']['data']).decode('utf-8')
    message_data = json.loads(pubsub_message)
    
    # Process message
    user_id = message_data.get('user_id')
    action = message_data.get('action')
    
    if action == 'user_signup':
        send_welcome_email(user_id)
    elif action == 'order_placed':
        process_order(message_data)
```

**Firestore Trigger:**
```python
def on_user_create(cloud_event):
    """Triggered when a user document is created."""
    from google.cloud import firestore
    
    # Get document data
    document_path = cloud_event.data['value']['name']
    user_data = cloud_event.data['value']['fields']
    
    # Extract user info
    email = user_data['email']['stringValue']
    name = user_data['name']['stringValue']
    
    # Initialize user profile
    db = firestore.Client()
    profile_ref = db.collection('profiles').document(document_path.split('/')[-1])
    profile_ref.set({
        'email': email,
        'name': name,
        'created_at': firestore.SERVER_TIMESTAMP,
        'preferences': {},
        'settings': {
            'notifications': True,
            'theme': 'light'
        }
    })
```

---

## 🔹 Product Versions

### 1st Generation vs 2nd Generation

| Feature | 1st Generation | 2nd Generation |
|---------|----------------|----------------|
| **Infrastructure** | App Engine | Cloud Run + Eventarc |
| **Concurrency** | 1 request/instance | Up to 1000 requests/instance |
| **Event Sources** | 7 trigger types | 90+ event sources |
| **Max Memory** | 8 GiB | 32 GiB |
| **Max CPU** | 2 vCPU | 8 vCPU |
| **Max Timeout** | 9 minutes | 60 minutes (HTTP), 9 minutes (event) |
| **Min Instances** | 0 | 0-1000 |
| **Traffic Splitting** | ❌ | ✅ |
| **Revisions** | ❌ | ✅ |
| **Service Account** | App Engine default | Compute Engine default |

### Migration Considerations

**When to Use 1st Gen:**
- Simple, lightweight functions
- Low traffic applications
- Cost-sensitive workloads
- Existing 1st gen functions working well

**When to Use 2nd Gen (Recommended):**
- High-traffic applications
- CPU-intensive workloads
- Need for concurrency
- Advanced event sources
- Traffic management requirements

**Migration Strategy:**
```bash
# Deploy 2nd gen alongside 1st gen
gcloud functions deploy my-function-v2 \
    --gen2 \
    --runtime python311 \
    --trigger-http \
    --source .

# Test thoroughly
curl https://us-central1-project.cloudfunctions.net/my-function-v2

# Gradually migrate traffic
# Update client applications to use new endpoint

# Retire 1st gen function
gcloud functions delete my-function-v1
```

### 2nd Generation Benefits

**Enhanced Performance:**
```yaml
# 2nd gen configuration
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-function
  annotations:
    run.googleapis.com/execution-environment: gen2
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "100"
        autoscaling.knative.dev/minScale: "1"
        run.googleapis.com/cpu-throttling: "false"
    spec:
      containerConcurrency: 1000
      containers:
      - image: gcr.io/project/function
        resources:
          limits:
            cpu: "4"
            memory: "16Gi"
```

**Advanced Event Handling:**
```python
# Eventarc integration (2nd gen)
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def process_audit_log(cloud_event: CloudEvent):
    """Process Cloud Audit Log events."""
    
    # Access rich event metadata
    event_type = cloud_event.get_type()
    source = cloud_event.get_source()
    subject = cloud_event.get_subject()
    
    # Process different audit events
    if 'storage.googleapis.com' in source:
        handle_storage_audit(cloud_event.data)
    elif 'compute.googleapis.com' in source:
        handle_compute_audit(cloud_event.data)
```

---

## 🔹 Function Examples

### HTTP Function Examples

**Python Flask-style:**
```python
import functions_framework
from flask import Request
import json

@functions_framework.http
def api_endpoint(request: Request):
    """HTTP Cloud Function for API endpoint."""
    
    # Handle CORS
    if request.method == 'OPTIONS':
        headers = {
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
            'Access-Control-Allow-Headers': 'Content-Type, Authorization'
        }
        return ('', 204, headers)
    
    # Set CORS headers
    headers = {'Access-Control-Allow-Origin': '*'}
    
    try:
        if request.method == 'GET':
            # Get query parameters
            user_id = request.args.get('user_id')
            return (json.dumps({'user_id': user_id, 'status': 'active'}), 200, headers)
        
        elif request.method == 'POST':
            # Process JSON payload
            request_json = request.get_json(silent=True)
            if not request_json:
                return (json.dumps({'error': 'Invalid JSON'}), 400, headers)
            
            # Business logic
            result = process_user_data(request_json)
            return (json.dumps(result), 201, headers)
    
    except Exception as e:
        return (json.dumps({'error': str(e)}), 500, headers)

def process_user_data(data):
    """Process user data and return result."""
    # Validate required fields
    required_fields = ['name', 'email']
    for field in required_fields:
        if field not in data:
            raise ValueError(f"Missing required field: {field}")
    
    # Simulate processing
    return {
        'id': generate_user_id(),
        'name': data['name'],
        'email': data['email'],
        'created_at': datetime.utcnow().isoformat()
    }
```

**Node.js Express-style:**
```javascript
const functions = require('@google-cloud/functions-framework');

functions.http('apiEndpoint', (req, res) => {
  // Set CORS headers
  res.set('Access-Control-Allow-Origin', '*');
  
  if (req.method === 'OPTIONS') {
    res.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.set('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.status(204).send('');
    return;
  }

  try {
    switch (req.method) {
      case 'GET':
        const userId = req.query.user_id;
        res.json({ user_id: userId, status: 'active' });
        break;
        
      case 'POST':
        const userData = req.body;
        if (!userData.name || !userData.email) {
          res.status(400).json({ error: 'Missing required fields' });
          return;
        }
        
        const result = {
          id: generateUserId(),
          name: userData.name,
          email: userData.email,
          created_at: new Date().toISOString()
        };
        
        res.status(201).json(result);
        break;
        
      default:
        res.status(405).json({ error: 'Method not allowed' });
    }
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: error.message });
  }
});

function generateUserId() {
  return Math.random().toString(36).substr(2, 9);
}
```

### Pub/Sub Function Examples

**Message Processing:**
```python
import functions_framework
import base64
import json
from google.cloud import firestore, storage

@functions_framework.cloud_event
def process_order(cloud_event):
    """Process order messages from Pub/Sub."""
    
    # Decode Pub/Sub message
    message_data = base64.b64decode(cloud_event.data['message']['data'])
    order_data = json.loads(message_data.decode('utf-8'))
    
    # Validate order data
    required_fields = ['order_id', 'user_id', 'items', 'total']
    for field in required_fields:
        if field not in order_data:
            print(f"Invalid order: missing {field}")
            return
    
    try:
        # Process order
        order_id = order_data['order_id']
        
        # Update inventory
        update_inventory(order_data['items'])
        
        # Send confirmation email
        send_order_confirmation(order_data)
        
        # Update order status
        update_order_status(order_id, 'processed')
        
        print(f"Successfully processed order {order_id}")
        
    except Exception as e:
        print(f"Error processing order {order_data.get('order_id', 'unknown')}: {e}")
        # Send to dead letter queue or retry logic
        handle_processing_error(order_data, str(e))

def update_inventory(items):
    """Update inventory levels."""
    db = firestore.Client()
    
    for item in items:
        product_id = item['product_id']
        quantity = item['quantity']
        
        # Update in transaction
        @firestore.transactional
        def update_stock(transaction, product_ref):
            product = product_ref.get(transaction=transaction)
            if product.exists:
                current_stock = product.get('stock')
                new_stock = current_stock - quantity
                if new_stock < 0:
                    raise ValueError(f"Insufficient stock for product {product_id}")
                transaction.update(product_ref, {'stock': new_stock})
        
        product_ref = db.collection('products').document(product_id)
        transaction = db.transaction()
        update_stock(transaction, product_ref)
```

**Batch Processing:**
```python
@functions_framework.cloud_event
def batch_process_logs(cloud_event):
    """Process log entries in batches."""
    from google.cloud import bigquery, logging
    
    # Get batch of log entries
    message_data = json.loads(base64.b64decode(cloud_event.data['message']['data']))
    batch_id = message_data['batch_id']
    
    # Query logs
    logging_client = logging.Client()
    filter_str = f'timestamp>="{message_data["start_time"]}" AND timestamp<"{message_data["end_time"]}"'
    
    entries = list(logging_client.list_entries(filter_=filter_str))
    
    if not entries:
        print(f"No log entries found for batch {batch_id}")
        return
    
    # Transform logs for BigQuery
    rows = []
    for entry in entries:
        rows.append({
            'timestamp': entry.timestamp.isoformat(),
            'severity': entry.severity,
            'log_name': entry.log_name,
            'resource_type': entry.resource.type,
            'message': str(entry.payload),
            'batch_id': batch_id
        })
    
    # Insert into BigQuery
    bq_client = bigquery.Client()
    table_ref = bq_client.dataset('logs').table('processed_logs')
    
    errors = bq_client.insert_rows_json(table_ref, rows)
    if errors:
        print(f"BigQuery insert errors: {errors}")
    else:
        print(f"Successfully processed {len(rows)} log entries for batch {batch_id}")
```

---

## 🔹 Scaling & Concurrency

### Scaling Behavior

**1st Generation Scaling:**
- **One request per instance** - Horizontal scaling only
- **Cold start** for each new instance
- **Automatic scaling** based on incoming requests
- **No minimum instances** - scales to zero

**2nd Generation Scaling:**
- **Up to 1000 concurrent requests** per instance
- **Configurable concurrency** (1-1000)
- **Minimum instances** support (0-1000)
- **Better resource utilization**

### Concurrency Configuration

**Optimal Concurrency Settings:**
```bash
# CPU-bound workloads (lower concurrency)
gcloud functions deploy cpu-intensive \
    --gen2 \
    --runtime python311 \
    --trigger-http \
    --concurrency 10 \
    --memory 2Gi \
    --cpu 2

# I/O-bound workloads (higher concurrency)  
gcloud functions deploy io-intensive \
    --gen2 \
    --runtime python311 \
    --trigger-http \
    --concurrency 1000 \
    --memory 512Mi \
    --cpu 1
```

**Concurrency Best Practices:**
```python
import asyncio
import aiohttp
from concurrent.futures import ThreadPoolExecutor

# For I/O-bound operations
async def async_http_function(request):
    """Async function for high concurrency."""
    
    async with aiohttp.ClientSession() as session:
        tasks = []
        for url in get_urls_to_process():
            tasks.append(fetch_data(session, url))
        
        results = await asyncio.gather(*tasks)
        return process_results(results)

async def fetch_data(session, url):
    async with session.get(url) as response:
        return await response.json()

# For CPU-bound operations
def cpu_bound_function(request):
    """CPU-intensive function with thread pool."""
    
    with ThreadPoolExecutor(max_workers=4) as executor:
        futures = []
        for data_chunk in get_data_chunks():
            futures.append(executor.submit(process_chunk, data_chunk))
        
        results = [future.result() for future in futures]
        return combine_results(results)
```

### Cold Start Mitigation

**Minimize Cold Starts:**
```bash
# Set minimum instances (costs more but reduces latency)
gcloud functions deploy warm-function \
    --gen2 \
    --runtime python311 \
    --trigger-http \
    --min-instances 1 \
    --max-instances 100
```

**Optimize Function Code:**
```python
# Global variables for connection reuse
import os
from google.cloud import firestore

# Initialize outside function (reused across invocations)
db_client = None

def get_db_client():
    global db_client
    if db_client is None:
        db_client = firestore.Client()
    return db_client

@functions_framework.http
def optimized_function(request):
    """Optimized function with connection reuse."""
    
    # Reuse existing connection
    db = get_db_client()
    
    # Lazy import heavy libraries
    if request.path == '/ml-predict':
        import tensorflow as tf  # Only import when needed
        return ml_prediction(request, tf)
    
    # Regular processing
    return regular_processing(request, db)
```

---

## 🔹 Deployment with gcloud

### Basic Deployment Commands

**HTTP Function Deployment:**
```bash
# Basic HTTP function
gcloud functions deploy my-http-function \
    --gen2 \
    --runtime python311 \
    --region us-central1 \
    --source . \
    --entry-point main \
    --trigger-http \
    --allow-unauthenticated

# With custom configuration
gcloud functions deploy my-api \
    --gen2 \
    --runtime python311 \
    --region us-central1 \
    --source . \
    --memory 1Gi \
    --cpu 2 \
    --timeout 300s \
    --concurrency 100 \
    --min-instances 1 \
    --max-instances 50 \
    --trigger-http \
    --set-env-vars "DATABASE_URL=postgresql://...,LOG_LEVEL=info" \
    --set-secrets "API_KEY=external-api-key:latest"
```

**Event-Driven Function Deployment:**
```bash
# Pub/Sub trigger
gcloud functions deploy process-messages \
    --gen2 \
    --runtime python311 \
    --region us-central1 \
    --source . \
    --trigger-topic order-events \
    --memory 512Mi \
    --timeout 60s

# Cloud Storage trigger
gcloud functions deploy process-images \
    --gen2 \
    --runtime python311 \
    --region us-central1 \
    --source . \
    --trigger-bucket my-images-bucket \
    --trigger-event google.storage.object.finalize

# Firestore trigger
gcloud functions deploy on-user-create \
    --gen2 \
    --runtime python311 \
    --region us-central1 \
    --source . \
    --trigger-event providers/cloud.firestore/eventTypes/document.create \
    --trigger-resource "projects/my-project/databases/(default)/documents/users/{userId}"
```

### Advanced Deployment Options

**Source Code Management:**
```bash
# Deploy from Cloud Storage
gcloud functions deploy my-function \
    --gen2 \
    --runtime python311 \
    --source gs://my-source-bucket/function-source.zip \
    --trigger-http

# Deploy from Git repository
gcloud functions deploy my-function \
    --gen2 \
    --runtime python311 \
    --source-url https://github.com/myorg/my-functions \
    --source-branch main \
    --source-dir functions/my-function \
    --trigger-http
```

**Service Account Configuration:**
```bash
# Create dedicated service account
gcloud iam service-accounts create function-sa \
    --display-name "Function Service Account"

# Grant necessary permissions
gcloud projects add-iam-policy-binding my-project \
    --member "serviceAccount:function-sa@my-project.iam.gserviceaccount.com" \
    --role "roles/firestore.user"

# Deploy with service account
gcloud functions deploy secure-function \
    --gen2 \
    --runtime python311 \
    --trigger-http \
    --service-account function-sa@my-project.iam.gserviceaccount.com
```

**Container Registry Configuration:**
```bash
# Use Artifact Registry for function images
gcloud functions deploy my-function \
    --gen2 \
    --runtime python311 \
    --trigger-http \
    --docker-repository projects/my-project/locations/us-central1/repositories/functions
```

### Deployment Automation

**Cloud Build Integration:**
```yaml
# cloudbuild.yaml
steps:
# Run tests
- name: 'python:3.11'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    pip install -r requirements.txt
    python -m pytest tests/

# Deploy function
- name: 'gcr.io/cloud-builders/gcloud'
  args:
  - 'functions'
  - 'deploy'
  - 'my-function'
  - '--gen2'
  - '--runtime=python311'
  - '--region=us-central1'
  - '--source=.'
  - '--trigger-http'
  - '--memory=512Mi'
  - '--timeout=60s'

# Integration test
- name: 'gcr.io/cloud-builders/curl'
  args: ['https://us-central1-my-project.cloudfunctions.net/my-function']
```

**GitHub Actions Deployment:**
```yaml
# .github/workflows/deploy-functions.yml
name: Deploy Cloud Functions
on:
  push:
    branches: [main]
    paths: ['functions/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v0
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ secrets.GCP_PROJECT_ID }}
    
    - name: Deploy Functions
      run: |
        for dir in functions/*/; do
          if [ -f "$dir/main.py" ]; then
            function_name=$(basename "$dir")
            echo "Deploying $function_name"
            
            gcloud functions deploy $function_name \
              --gen2 \
              --runtime python311 \
              --region us-central1 \
              --source "$dir" \
              --trigger-http \
              --quiet
          fi
        done
```

---

## 🔹 Best Practices

### Security Best Practices

**Secret Management:**
```python
from google.cloud import secretmanager

def get_secret(secret_name, version="latest"):
    """Retrieve secret from Secret Manager."""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{os.environ['GCP_PROJECT']}/secrets/{secret_name}/versions/{version}"
    
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

@functions_framework.http
def secure_function(request):
    """Function using secrets securely."""
    
    # Get API key from Secret Manager
    api_key = get_secret("external-api-key")
    
    # Use secret in API call
    headers = {"Authorization": f"Bearer {api_key}"}
    response = requests.get("https://api.example.com/data", headers=headers)
    
    return response.json()
```

**IAM Configuration:**
```bash
# Create function-specific service account
gcloud iam service-accounts create order-processor-sa

# Grant minimal permissions
gcloud projects add-iam-policy-binding my-project \
    --member "serviceAccount:order-processor-sa@my-project.iam.gserviceaccount.com" \
    --role "roles/firestore.user"

gcloud projects add-iam-policy-binding my-project \
    --member "serviceAccount:order-processor-sa@my-project.iam.gserviceaccount.com" \
    --role "roles/pubsub.publisher"

# Deploy with restricted permissions
gcloud functions deploy process-orders \
    --service-account order-processor-sa@my-project.iam.gserviceaccount.com
```

### Performance Optimization

**Dependency Management:**
```python
# requirements.txt - Keep minimal
functions-framework==3.4.0
google-cloud-firestore==2.11.1
requests==2.31.0

# Avoid heavy dependencies
# tensorflow  # Only if absolutely necessary
# pandas      # Use built-in libraries when possible
```

**Connection Pooling:**
```python
import sqlalchemy
from sqlalchemy.pool import QueuePool

# Global connection pool (reused across invocations)
engine = None

def get_db_engine():
    global engine
    if engine is None:
        engine = sqlalchemy.create_engine(
            os.environ['DATABASE_URL'],
            poolclass=QueuePool,
            pool_size=1,  # Small pool for functions
            max_overflow=0,
            pool_pre_ping=True
        )
    return engine

@functions_framework.http
def db_function(request):
    """Function with optimized database connections."""
    engine = get_db_engine()
    
    with engine.connect() as conn:
        result = conn.execute("SELECT * FROM users LIMIT 10")
        return {"users": [dict(row) for row in result]}
```

**Caching Strategies:**
```python
import functools
import time

# In-memory cache (persists across invocations)
cache = {}

def cached_function(ttl_seconds=300):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # Create cache key
            key = f"{func.__name__}:{hash(str(args) + str(kwargs))}"
            
            # Check cache
            if key in cache:
                result, timestamp = cache[key]
                if time.time() - timestamp < ttl_seconds:
                    return result
            
            # Execute function and cache result
            result = func(*args, **kwargs)
            cache[key] = (result, time.time())
            return result
        return wrapper
    return decorator

@cached_function(ttl_seconds=600)
def expensive_computation(data):
    """Expensive function with caching."""
    # Simulate expensive operation
    time.sleep(2)
    return {"processed": len(data), "timestamp": time.time()}
```

### Error Handling & Monitoring

**Structured Logging:**
```python
import logging
import json
from google.cloud import logging as cloud_logging

# Setup Cloud Logging
cloud_logging.Client().setup_logging()

def structured_log(level, message, **kwargs):
    """Log with structured data."""
    log_entry = {
        "message": message,
        "timestamp": time.time(),
        **kwargs
    }
    
    if level == "ERROR":
        logging.error(json.dumps(log_entry))
    elif level == "WARNING":
        logging.warning(json.dumps(log_entry))
    else:
        logging.info(json.dumps(log_entry))

@functions_framework.http
def monitored_function(request):
    """Function with comprehensive monitoring."""
    start_time = time.time()
    
    try:
        # Log request start
        structured_log("INFO", "Function started", 
                      function_name="monitored_function",
                      request_id=request.headers.get('X-Request-ID'))
        
        # Process request
        result = process_request(request)
        
        # Log success
        duration = time.time() - start_time
        structured_log("INFO", "Function completed successfully",
                      function_name="monitored_function",
                      duration_ms=duration * 1000,
                      status="success")
        
        return result
        
    except Exception as e:
        # Log error with context
        duration = time.time() - start_time
        structured_log("ERROR", "Function failed",
                      function_name="monitored_function",
                      error=str(e),
                      error_type=type(e).__name__,
                      duration_ms=duration * 1000,
                      status="error")
        
        # Return error response
        return {"error": "Internal server error"}, 500
```

**Dead Letter Queue Pattern:**
```python
from google.cloud import pubsub_v1

def process_with_dlq(cloud_event):
    """Process message with dead letter queue fallback."""
    
    try:
        # Attempt to process message
        message_data = json.loads(base64.b64decode(cloud_event.data['message']['data']))
        result = process_message(message_data)
        
        # Log success
        print(f"Successfully processed message: {message_data.get('id')}")
        return result
        
    except Exception as e:
        # Send to dead letter queue
        publisher = pubsub_v1.PublisherClient()
        dlq_topic = publisher.topic_path(os.environ['GCP_PROJECT'], 'failed-messages')
        
        error_message = {
            "original_message": cloud_event.data,
            "error": str(e),
            "error_type": type(e).__name__,
            "timestamp": time.time(),
            "function_name": "process_with_dlq"
        }
        
        publisher.publish(dlq_topic, json.dumps(error_message).encode())
        
        # Log error
        print(f"Message sent to DLQ: {e}")
        raise  # Re-raise to trigger Pub/Sub retry if desired
```

---

## 🔹 Architecture Patterns

### Event-Driven Architecture

**Microservices with Functions:**
```
User Action → API Gateway → Cloud Function → Pub/Sub → Processing Functions
                                ↓
                          Cloud Firestore ← Notification Function ← Pub/Sub
```

**Implementation Example:**
```python
# API Gateway Function
@functions_framework.http
def api_gateway(request):
    """Main API gateway function."""
    
    path = request.path
    method = request.method
    
    # Route to appropriate handler
    if path.startswith('/users'):
        return handle_users(request)
    elif path.startswith('/orders'):
        return handle_orders(request)
    else:
        return {"error": "Not found"}, 404

def handle_orders(request):
    """Handle order-related requests."""
    from google.cloud import pubsub_v1
    
    if request.method == 'POST':
        order_data = request.get_json()
        
        # Validate order
        if not validate_order(order_data):
            return {"error": "Invalid order data"}, 400
        
        # Publish to processing queue
        publisher = pubsub_v1.PublisherClient()
        topic_path = publisher.topic_path(PROJECT_ID, 'order-processing')
        
        message_data = json.dumps(order_data).encode()
        publisher.publish(topic_path, message_data)
        
        return {"status": "Order queued for processing", "order_id": order_data['id']}

# Order Processing Function
@functions_framework.cloud_event
def process_order(cloud_event):
    """Process order from Pub/Sub."""
    
    order_data = json.loads(base64.b64decode(cloud_event.data['message']['data']))
    
    # Process payment
    payment_result = process_payment(order_data)
    
    if payment_result['success']:
        # Update inventory
        update_inventory(order_data['items'])
        
        # Send confirmation
        send_confirmation_email(order_data)
        
        # Publish completion event
        publish_order_completed(order_data['id'])
    else:
        # Handle payment failure
        handle_payment_failure(order_data, payment_result['error'])
```

### Serverless Data Pipeline

**ETL Pipeline with Functions:**
```
Data Source → Cloud Storage → Function (Extract) → Pub/Sub → Function (Transform) → BigQuery
                                                      ↓
                                              Function (Load) → Cloud SQL
```

**Pipeline Implementation:**
```python
# Extract Function (Storage Trigger)
@functions_framework.cloud_event
def extract_data(cloud_event):
    """Extract data from uploaded files."""
    from google.cloud import storage, pubsub_v1
    
    bucket_name = cloud_event.data['bucket']
    file_name = cloud_event.data['name']
    
    # Download and parse file
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(file_name)
    
    content = blob.download_as_text()
    records = parse_csv_content(content)
    
    # Publish records for transformation
    publisher = pubsub_v1.PublisherClient()
    topic_path = publisher.topic_path(PROJECT_ID, 'raw-data')
    
    for record in records:
        message_data = json.dumps(record).encode()
        publisher.publish(topic_path, message_data)
    
    print(f"Extracted {len(records)} records from {file_name}")

# Transform Function (Pub/Sub Trigger)
@functions_framework.cloud_event
def transform_data(cloud_event):
    """Transform raw data records."""
    
    raw_record = json.loads(base64.b64decode(cloud_event.data['message']['data']))
    
    # Apply transformations
    transformed_record = {
        'id': raw_record['id'],
        'timestamp': parse_timestamp(raw_record['date']),
        'amount': float(raw_record['amount']),
        'category': normalize_category(raw_record['category']),
        'processed_at': datetime.utcnow().isoformat()
    }
    
    # Validate transformed record
    if validate_record(transformed_record):
        # Send to loading queue
        publisher = pubsub_v1.PublisherClient()
        topic_path = publisher.topic_path(PROJECT_ID, 'transformed-data')
        
        message_data = json.dumps(transformed_record).encode()
        publisher.publish(topic_path, message_data)
    else:
        # Send to error queue
        send_to_error_queue(raw_record, "Validation failed")

# Load Function (Pub/Sub Trigger)
@functions_framework.cloud_event
def load_data(cloud_event):
    """Load transformed data to BigQuery."""
    from google.cloud import bigquery
    
    record = json.loads(base64.b64decode(cloud_event.data['message']['data']))
    
    # Insert into BigQuery
    client = bigquery.Client()
    table_ref = client.dataset('analytics').table('processed_records')
    
    errors = client.insert_rows_json(table_ref, [record])
    
    if errors:
        print(f"BigQuery insert errors: {errors}")
        send_to_error_queue(record, f"BigQuery error: {errors}")
    else:
        print(f"Successfully loaded record {record['id']}")
```

---

## 🔹 DevOps Integration

### CI/CD Pipeline Integration

**Multi-Environment Deployment:**
```yaml
# .github/workflows/functions-cicd.yml
name: Functions CI/CD
on:
  push:
    branches: [main, develop]
    paths: ['functions/**']

env:
  PROJECT_ID_DEV: my-project-dev
  PROJECT_ID_PROD: my-project-prod

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Python
      uses: actions/setup-python@v2
      with:
        python-version: 3.11
    
    - name: Install dependencies
      run: |
        pip install -r functions/requirements.txt
        pip install pytest pytest-cov
    
    - name: Run tests
      run: |
        cd functions
        pytest tests/ --cov=. --cov-report=xml
    
    - name: Upload coverage
      uses: codecov/codecov-action@v1

  deploy-dev:
    needs: test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v0
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY_DEV }}
        project_id: ${{ env.PROJECT_ID_DEV }}
    
    - name: Deploy to Development
      run: |
        cd functions
        gcloud functions deploy api-function \
          --gen2 \
          --runtime python311 \
          --region us-central1 \
          --source . \
          --trigger-http \
          --set-env-vars "ENVIRONMENT=development"

  deploy-prod:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v0
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY_PROD }}
        project_id: ${{ env.PROJECT_ID_PROD }}
    
    - name: Deploy to Production
      run: |
        cd functions
        gcloud functions deploy api-function \
          --gen2 \
          --runtime python311 \
          --region us-central1 \
          --source . \
          --trigger-http \
          --min-instances 1 \
          --max-instances 100 \
          --set-env-vars "ENVIRONMENT=production"
```

### Infrastructure as Code

**Terraform Configuration:**
```hcl
# functions.tf
resource "google_cloudfunctions2_function" "api_function" {
  name        = "api-function"
  location    = "us-central1"
  description = "Main API function"

  build_config {
    runtime     = "python311"
    entry_point = "main"
    
    source {
      storage_source {
        bucket = google_storage_bucket.function_source.name
        object = google_storage_bucket_object.function_source.name
      }
    }
  }

  service_config {
    max_instance_count = 100
    min_instance_count = 1
    available_memory   = "512Mi"
    timeout_seconds    = 60
    
    environment_variables = {
      ENVIRONMENT = var.environment
      PROJECT_ID  = var.project_id
    }
    
    secret_environment_variables {
      key        = "DATABASE_PASSWORD"
      project_id = var.project_id
      secret     = "database-password"
      version    = "latest"
    }
    
    service_account_email = google_service_account.function_sa.email
  }

  event_trigger {
    trigger_region = "us-central1"
    event_type     = "google.cloud.pubsub.topic.v1.messagePublished"
    pubsub_topic   = google_pubsub_topic.function_trigger.id
  }
}

resource "google_service_account" "function_sa" {
  account_id   = "function-service-account"
  display_name = "Function Service Account"
}

resource "google_project_iam_member" "function_sa_roles" {
  for_each = toset([
    "roles/firestore.user",
    "roles/pubsub.publisher",
    "roles/secretmanager.secretAccessor"
  ])
  
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.function_sa.email}"
}
```

### Monitoring & Alerting

**Cloud Monitoring Setup:**
```yaml
# monitoring.yaml
alertPolicy:
  displayName: "Cloud Function Error Rate"
  conditions:
  - displayName: "Error rate > 5%"
    conditionThreshold:
      filter: 'resource.type="cloud_function" resource.label.function_name="api-function"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 0.05
      duration: 300s
      aggregations:
      - alignmentPeriod: 60s
        perSeriesAligner: ALIGN_RATE
        crossSeriesReducer: REDUCE_MEAN
        groupByFields:
        - resource.label.function_name

  - displayName: "High latency"
    conditionThreshold:
      filter: 'resource.type="cloud_function" resource.label.function_name="api-function"'
      comparison: COMPARISON_GREATER_THAN
      thresholdValue: 5000  # 5 seconds
      duration: 180s

notificationChannels:
- type: "slack"
  labels:
    channel_name: "#alerts"
- type: "email"
  labels:
    email_address: "devops@company.com"
```

**Custom Metrics:**
```python
from google.cloud import monitoring_v3
import time

def record_custom_metric(metric_name, value, labels=None):
    """Record custom metric to Cloud Monitoring."""
    
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{os.environ['GCP_PROJECT']}"
    
    series = monitoring_v3.TimeSeries()
    series.metric.type = f"custom.googleapis.com/function/{metric_name}"
    series.resource.type = "cloud_function"
    series.resource.labels["function_name"] = os.environ.get('FUNCTION_NAME', 'unknown')
    series.resource.labels["region"] = os.environ.get('FUNCTION_REGION', 'us-central1')
    
    # Add custom labels
    if labels:
        for key, value in labels.items():
            series.metric.labels[key] = str(value)
    
    # Add data point
    point = series.points.add()
    point.value.double_value = float(value)
    point.interval.end_time.seconds = int(time.time())
    
    # Write to Cloud Monitoring
    client.create_time_series(name=project_name, time_series=[series])

@functions_framework.http
def monitored_api_function(request):
    """Function with custom metrics."""
    start_time = time.time()
    
    try:
        # Process request
        result = process_api_request(request)
        
        # Record success metrics
        duration = time.time() - start_time
        record_custom_metric("request_duration", duration * 1000, {"status": "success"})
        record_custom_metric("request_count", 1, {"status": "success", "endpoint": request.path})
        
        return result
        
    except Exception as e:
        # Record error metrics
        duration = time.time() - start_time
        record_custom_metric("request_duration", duration * 1000, {"status": "error"})
        record_custom_metric("request_count", 1, {"status": "error", "error_type": type(e).__name__})
        
        raise
```

---

## 🔹 Monitoring & Troubleshooting

### Logging & Debugging

**Structured Logging Best Practices:**
```python
import logging
import json
import traceback
from google.cloud import logging as cloud_logging

# Setup Cloud Logging
cloud_logging.Client().setup_logging()

class StructuredLogger:
    def __init__(self, function_name):
        self.function_name = function_name
        self.logger = logging.getLogger(function_name)
    
    def log(self, level, message, **kwargs):
        log_entry = {
            "message": message,
            "function_name": self.function_name,
            "timestamp": time.time(),
            **kwargs
        }
        
        if level == "ERROR":
            self.logger.error(json.dumps(log_entry))
        elif level == "WARNING":
            self.logger.warning(json.dumps(log_entry))
        else:
            self.logger.info(json.dumps(log_entry))
    
    def log_request(self, request, response_status=None, duration=None):
        """Log HTTP request details."""
        self.log("INFO", "HTTP request processed",
                request_method=request.method,
                request_path=request.path,
                request_id=request.headers.get('X-Request-ID'),
                user_agent=request.headers.get('User-Agent'),
                response_status=response_status,
                duration_ms=duration)
    
    def log_error(self, error, context=None):
        """Log error with full context."""
        self.log("ERROR", "Function error occurred",
                error_message=str(error),
                error_type=type(error).__name__,
                traceback=traceback.format_exc(),
                context=context or {})

# Usage in function
logger = StructuredLogger("api-function")

@functions_framework.http
def api_function(request):
    start_time = time.time()
    request_id = request.headers.get('X-Request-ID', 'unknown')
    
    try:
        logger.log("INFO", "Function started", request_id=request_id)
        
        # Process request
        result = process_request(request)
        
        # Log success
        duration = (time.time() - start_time) * 1000
        logger.log_request(request, response_status=200, duration=duration)
        
        return result
        
    except Exception as e:
        duration = (time.time() - start_time) * 1000
        logger.log_error(e, context={
            "request_id": request_id,
            "request_method": request.method,
            "request_path": request.path,
            "duration_ms": duration
        })
        
        return {"error": "Internal server error"}, 500
```

### Common Issues & Solutions

**1. Cold Start Performance:**
```python
# Problem: Slow cold starts
import time

# Solution: Optimize imports and initialization
# Global initialization (runs once per instance)
start_time = time.time()

# Lazy import heavy libraries
_ml_model = None
_db_connection = None

def get_ml_model():
    global _ml_model
    if _ml_model is None:
        import tensorflow as tf  # Heavy import
        _ml_model = tf.keras.models.load_model('model.h5')
    return _ml_model

def get_db_connection():
    global _db_connection
    if _db_connection is None:
        import psycopg2
        _db_connection = psycopg2.connect(os.environ['DATABASE_URL'])
    return _db_connection

print(f"Global initialization took {(time.time() - start_time) * 1000:.2f}ms")

@functions_framework.http
def optimized_function(request):
    # Fast path for health checks
    if request.path == '/health':
        return 'OK'
    
    # Only initialize heavy resources when needed
    if request.path == '/predict':
        model = get_ml_model()
        return make_prediction(model, request.get_json())
    
    if request.path == '/data':
        db = get_db_connection()
        return fetch_data(db, request.args)
```

**2. Memory Issues:**
```python
# Problem: Memory leaks and high usage
import gc
import psutil
import os

def monitor_memory():
    """Monitor memory usage and trigger cleanup if needed."""
    process = psutil.Process(os.getpid())
    memory_mb = process.memory_info().rss / 1024 / 1024
    
    # Log memory usage
    print(f"Memory usage: {memory_mb:.2f} MB")
    
    # Trigger cleanup if memory usage is high
    if memory_mb > 400:  # 400MB threshold
        print("High memory usage detected, triggering garbage collection")
        gc.collect()
        
        # Check memory after cleanup
        new_memory_mb = process.memory_info().rss / 1024 / 1024
        print(f"Memory after cleanup: {new_memory_mb:.2f} MB")
    
    return memory_mb

@functions_framework.http
def memory_conscious_function(request):
    """Function with memory monitoring."""
    
    try:
        # Monitor memory at start
        initial_memory = monitor_memory()
        
        # Process request
        result = process_large_dataset(request.get_json())
        
        # Monitor memory after processing
        final_memory = monitor_memory()
        
        print(f"Memory delta: {final_memory - initial_memory:.2f} MB")
        
        return result
        
    finally:
        # Always cleanup
        gc.collect()
```

**3. Timeout Issues:**
```python
# Problem: Function timeouts
import signal
import asyncio
from concurrent.futures import ThreadPoolExecutor, TimeoutError

class TimeoutHandler:
    def __init__(self, timeout_seconds):
        self.timeout_seconds = timeout_seconds
    
    def __enter__(self):
        signal.signal(signal.SIGALRM, self._timeout_handler)
        signal.alarm(self.timeout_seconds)
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        signal.alarm(0)
    
    def _timeout_handler(self, signum, frame):
        raise TimeoutError(f"Operation timed out after {self.timeout_seconds} seconds")

@functions_framework.http
def timeout_aware_function(request):
    """Function with timeout handling."""
    
    try:
        # Set timeout slightly less than function timeout
        with TimeoutHandler(55):  # 55 seconds for 60s function timeout
            result = process_long_running_task(request.get_json())
            return result
            
    except TimeoutError:
        # Handle timeout gracefully
        return {
            "error": "Request timed out",
            "message": "Operation took too long to complete"
        }, 408
    
    except Exception as e:
        return {"error": str(e)}, 500

async def async_timeout_function(request):
    """Async function with timeout."""
    
    try:
        # Use asyncio timeout
        result = await asyncio.wait_for(
            process_async_task(request.get_json()),
            timeout=55.0
        )
        return result
        
    except asyncio.TimeoutError:
        return {"error": "Request timed out"}, 408
```

**4. Concurrency Issues:**
```python
# Problem: Race conditions and shared state
import threading
from concurrent.futures import ThreadPoolExecutor
import queue

# Thread-safe counter
class ThreadSafeCounter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()
    
    def increment(self):
        with self._lock:
            self._value += 1
            return self._value
    
    def get_value(self):
        with self._lock:
            return self._value

# Global thread-safe counter
request_counter = ThreadSafeCounter()

@functions_framework.http
def concurrent_safe_function(request):
    """Function safe for high concurrency."""
    
    # Increment request counter safely
    request_num = request_counter.increment()
    
    # Use thread pool for CPU-bound work
    with ThreadPoolExecutor(max_workers=4) as executor:
        futures = []
        
        data_chunks = split_data(request.get_json())
        for chunk in data_chunks:
            future = executor.submit(process_chunk_safely, chunk, request_num)
            futures.append(future)
        
        # Collect results
        results = []
        for future in futures:
            try:
                result = future.result(timeout=30)
                results.append(result)
            except Exception as e:
                print(f"Chunk processing failed: {e}")
                results.append({"error": str(e)})
    
    return {
        "request_number": request_num,
        "results": results,
        "total_chunks": len(data_chunks)
    }

def process_chunk_safely(chunk, request_num):
    """Process data chunk in thread-safe manner."""
    
    # Use local variables and avoid global state
    local_result = {"chunk_id": chunk.get("id"), "request_num": request_num}
    
    try:
        # Process chunk
        processed_data = expensive_computation(chunk["data"])
        local_result["data"] = processed_data
        local_result["status"] = "success"
        
    except Exception as e:
        local_result["error"] = str(e)
        local_result["status"] = "error"
    
    return local_result
```

### Performance Monitoring

**Function Performance Metrics:**
```python
import time
import functools
from google.cloud import monitoring_v3

def performance_monitor(metric_prefix="function"):
    """Decorator to monitor function performance."""
    
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            start_time = time.time()
            
            try:
                result = func(*args, **kwargs)
                
                # Record success metrics
                duration = (time.time() - start_time) * 1000
                record_performance_metric(f"{metric_prefix}_duration", duration, {"status": "success"})
                record_performance_metric(f"{metric_prefix}_count", 1, {"status": "success"})
                
                return result
                
            except Exception as e:
                # Record error metrics
                duration = (time.time() - start_time) * 1000
                record_performance_metric(f"{metric_prefix}_duration", duration, {"status": "error"})
                record_performance_metric(f"{metric_prefix}_count", 1, {"status": "error", "error_type": type(e).__name__})
                
                raise
        
        return wrapper
    return decorator

def record_performance_metric(metric_name, value, labels=None):
    """Record performance metric to Cloud Monitoring."""
    
    try:
        client = monitoring_v3.MetricServiceClient()
        project_name = f"projects/{os.environ.get('GCP_PROJECT')}"
        
        series = monitoring_v3.TimeSeries()
        series.metric.type = f"custom.googleapis.com/{metric_name}"
        series.resource.type = "cloud_function"
        series.resource.labels["function_name"] = os.environ.get('FUNCTION_NAME', 'unknown')
        
        if labels:
            for key, value in labels.items():
                series.metric.labels[key] = str(value)
        
        point = series.points.add()
        point.value.double_value = float(value)
        point.interval.end_time.seconds = int(time.time())
        
        client.create_time_series(name=project_name, time_series=[series])
        
    except Exception as e:
        # Don't fail function if monitoring fails
        print(f"Failed to record metric {metric_name}: {e}")

# Usage
@performance_monitor("api_endpoint")
@functions_framework.http
def monitored_api_function(request):
    """API function with performance monitoring."""
    
    # Your function logic here
    return process_api_request(request)
```

This comprehensive Cloud Functions study guide provides you with the knowledge and practical examples needed to master Google Cloud Functions from a DevOps perspective. The guide covers everything from basic concepts to advanced deployment strategies, monitoring, and optimization techniques specifically tailored for your senior DevOps role.
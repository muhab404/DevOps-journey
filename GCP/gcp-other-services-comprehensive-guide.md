# ☁️ GCP Other Important Services - Comprehensive Guide for Senior DevOps Engineers
*Advanced Cloud Services, Integration Patterns, and Enterprise Architecture*

## Table of Contents
1. [Cloud Scheduler & Automation](#cloud-scheduler--automation)
2. [Development & Testing Tools](#development--testing-tools)
3. [DNS & Domain Management](#dns--domain-management)
4. [Cost Management & Planning](#cost-management--planning)
5. [Hybrid & Multi-Cloud Management](#hybrid--multi-cloud-management)
6. [API Management & Integration](#api-management--integration)
7. [Machine Learning & AI Platform](#machine-learning--ai-platform)
8. [Identity & Authentication Services](#identity--authentication-services)
9. [Event-Driven Architecture](#event-driven-architecture)
10. [Observability & Monitoring](#observability--monitoring)
11. [Service Discovery & Networking](#service-discovery--networking)
12. [Modern CLI & Storage Management](#modern-cli--storage-management)

---

## 🔹 Cloud Scheduler & Automation

### Cloud Scheduler Architecture

**Enterprise Scheduling Service:**
Cloud Scheduler is Google's fully managed, enterprise-grade cron service that enables reliable job scheduling across distributed systems.

**Core Components:**
- **Job Definition**: Cron expression, target, payload, retry configuration
- **Target Types**: HTTP endpoints, Pub/Sub topics, App Engine handlers
- **Execution Engine**: Distributed, fault-tolerant job execution
- **Monitoring Integration**: Cloud Logging, Cloud Monitoring integration

**Advanced Scheduling Patterns:**
```bash
# Complex cron expressions for enterprise scenarios
# Every weekday at 9 AM EST
0 9 * * 1-5

# Every 15 minutes during business hours
*/15 9-17 * * 1-5

# First day of every month at midnight
0 0 1 * *

# Every quarter (first day of Jan, Apr, Jul, Oct)
0 0 1 1,4,7,10 *
```

**Enterprise Use Cases:**

**Data Pipeline Orchestration:**
```bash
# Schedule ETL job via Pub/Sub
gcloud scheduler jobs create pubsub etl-daily-job \
  --schedule="0 2 * * *" \
  --topic=projects/PROJECT_ID/topics/etl-trigger \
  --message-body='{"pipeline":"daily-etl","date":"{{.timestamp}}"}' \
  --time-zone="America/New_York"
```

**Infrastructure Automation:**
```bash
# Schedule VM scaling operations
gcloud scheduler jobs create http vm-scale-down \
  --schedule="0 18 * * 1-5" \
  --uri="https://us-central1-PROJECT_ID.cloudfunctions.net/scaleDownVMs" \
  --http-method=POST \
  --headers="Content-Type=application/json" \
  --message-body='{"action":"scale_down","environment":"staging"}'
```

**Backup and Maintenance:**
```bash
# Schedule database backups
gcloud scheduler jobs create pubsub db-backup-job \
  --schedule="0 1 * * *" \
  --topic=projects/PROJECT_ID/topics/backup-trigger \
  --message-body='{"database":"production","type":"full_backup"}' \
  --max-retry-attempts=3 \
  --max-retry-duration=3600s
```

### Advanced Scheduler Configuration

**Retry and Error Handling:**
```bash
# Configure sophisticated retry logic
gcloud scheduler jobs create http api-health-check \
  --schedule="*/5 * * * *" \
  --uri="https://api.example.com/health" \
  --max-retry-attempts=5 \
  --max-retry-duration=1800s \
  --min-backoff-duration=30s \
  --max-backoff-duration=300s \
  --max-doublings=3
```

**Authentication and Security:**
```bash
# Schedule job with service account authentication
gcloud scheduler jobs create http secure-api-call \
  --schedule="0 */6 * * *" \
  --uri="https://secure-api.example.com/process" \
  --oidc-service-account-email=scheduler@PROJECT_ID.iam.gserviceaccount.com \
  --oidc-token-audience="https://secure-api.example.com"
```

**Integration with Cloud Functions:**
```python
# Cloud Function triggered by Cloud Scheduler
import functions_framework
from google.cloud import compute_v1
import json

@functions_framework.http
def scheduled_vm_management(request):
    """Manage VM instances based on schedule"""
    
    request_json = request.get_json()
    action = request_json.get('action')
    environment = request_json.get('environment')
    
    compute_client = compute_v1.InstancesClient()
    project_id = 'your-project-id'
    
    if action == 'scale_down':
        # Get instances with environment label
        instances = list_instances_by_label(compute_client, project_id, 
                                          'environment', environment)
        
        for instance in instances:
            if instance.status == 'RUNNING':
                # Stop non-production instances
                compute_client.stop(
                    project=project_id,
                    zone=extract_zone(instance.self_link),
                    instance=instance.name
                )
                
    elif action == 'scale_up':
        # Start instances for business hours
        instances = list_stopped_instances(compute_client, project_id, environment)
        for instance in instances:
            compute_client.start(
                project=project_id,
                zone=extract_zone(instance.self_link),
                instance=instance.name
            )
    
    return {'status': 'success', 'action': action, 'environment': environment}
```

---

## 🔹 Development & Testing Tools

### Cloud Emulators for Local Development

**Enterprise Development Workflow:**
Cloud Emulators enable local development and testing without cloud dependencies, crucial for CI/CD pipelines and developer productivity.

**Supported Emulators and Use Cases:**

**Cloud Firestore Emulator:**
```bash
# Start Firestore emulator
gcloud emulators firestore start --host-port=localhost:8080

# Configure application to use emulator
export FIRESTORE_EMULATOR_HOST=localhost:8080
export GOOGLE_CLOUD_PROJECT=test-project
```

**Cloud Pub/Sub Emulator:**
```bash
# Start Pub/Sub emulator
gcloud emulators pubsub start --host-port=localhost:8085

# Create topic and subscription in emulator
export PUBSUB_EMULATOR_HOST=localhost:8085
gcloud pubsub topics create test-topic
gcloud pubsub subscriptions create test-subscription --topic=test-topic
```

**Cloud Spanner Emulator:**
```bash
# Start Spanner emulator
gcloud emulators spanner start

# Configure connection
export SPANNER_EMULATOR_HOST=localhost:9010
```

**Docker-based Development Environment:**
```dockerfile
# Dockerfile for development environment
FROM google/cloud-sdk:alpine

# Install emulators
RUN gcloud components install cloud-firestore-emulator \
    cloud-pubsub-emulator \
    cloud-spanner-emulator

# Start script for multiple emulators
COPY start-emulators.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/start-emulators.sh

EXPOSE 8080 8085 9010
CMD ["/usr/local/bin/start-emulators.sh"]
```

**Integration Testing with Emulators:**
```python
# pytest configuration for emulator testing
import pytest
import subprocess
import time
from google.cloud import firestore
from google.cloud import pubsub_v1

@pytest.fixture(scope="session")
def firestore_emulator():
    """Start Firestore emulator for testing"""
    process = subprocess.Popen([
        'gcloud', 'emulators', 'firestore', 'start',
        '--host-port=localhost:8080'
    ])
    
    # Wait for emulator to start
    time.sleep(5)
    
    # Set environment variable
    os.environ['FIRESTORE_EMULATOR_HOST'] = 'localhost:8080'
    
    yield
    
    # Cleanup
    process.terminate()
    process.wait()

def test_firestore_operations(firestore_emulator):
    """Test Firestore operations using emulator"""
    client = firestore.Client(project='test-project')
    
    # Test document creation
    doc_ref = client.collection('users').document('test-user')
    doc_ref.set({'name': 'Test User', 'email': 'test@example.com'})
    
    # Test document retrieval
    doc = doc_ref.get()
    assert doc.exists
    assert doc.to_dict()['name'] == 'Test User'
```

---

## 🔹 DNS & Domain Management

### Cloud DNS Architecture

**Global DNS Infrastructure:**
Cloud DNS provides authoritative DNS serving from Google's global network, offering high availability and low latency worldwide.

**DNS Zone Types:**

**Public Zones:**
```bash
# Create public DNS zone
gcloud dns managed-zones create production-zone \
  --description="Production domain DNS zone" \
  --dns-name=example.com. \
  --visibility=public

# Add A record
gcloud dns record-sets transaction start --zone=production-zone
gcloud dns record-sets transaction add \
  --name=api.example.com. \
  --ttl=300 \
  --type=A \
  --zone=production-zone \
  "203.0.113.10"
gcloud dns record-sets transaction execute --zone=production-zone
```

**Private Zones:**
```bash
# Create private DNS zone for internal services
gcloud dns managed-zones create internal-zone \
  --description="Internal services DNS" \
  --dns-name=internal.example.com. \
  --visibility=private \
  --networks=projects/PROJECT_ID/global/networks/vpc-network

# Add internal service records
gcloud dns record-sets transaction start --zone=internal-zone
gcloud dns record-sets transaction add \
  --name=database.internal.example.com. \
  --ttl=300 \
  --type=A \
  --zone=internal-zone \
  "10.0.1.100"
gcloud dns record-sets transaction execute --zone=internal-zone
```

**Advanced DNS Configurations:**

**DNSSEC Implementation:**
```bash
# Enable DNSSEC for zone
gcloud dns managed-zones update production-zone \
  --dnssec-state=on

# Get DS records for domain registrar
gcloud dns managed-zones describe production-zone \
  --format="value(dnssecConfig.defaultKeySpecs[].keyType,dnssecConfig.defaultKeySpecs[].algorithm)"
```

**DNS Forwarding:**
```bash
# Create forwarding zone for hybrid DNS
gcloud dns managed-zones create onprem-forward \
  --description="Forward to on-premises DNS" \
  --dns-name=onprem.example.com. \
  --visibility=private \
  --networks=projects/PROJECT_ID/global/networks/vpc-network \
  --forwarding-targets=192.168.1.10,192.168.1.11
```

**DNS Policies:**
```bash
# Create DNS policy for alternative name servers
gcloud dns policies create custom-dns-policy \
  --description="Custom DNS policy for development" \
  --networks=projects/PROJECT_ID/global/networks/dev-network \
  --alternative-name-servers=8.8.8.8,8.8.4.4
```

### Enterprise DNS Patterns

**Multi-Environment DNS Strategy:**
```bash
# Production environment
api.example.com → 203.0.113.10
www.example.com → 203.0.113.20

# Staging environment  
api-staging.example.com → 203.0.113.30
www-staging.example.com → 203.0.113.40

# Development environment
api-dev.example.com → 203.0.113.50
www-dev.example.com → 203.0.113.60
```

**Geographic DNS Routing:**
```bash
# Create routing policy for global load balancing
gcloud dns record-sets create api.example.com. \
  --zone=production-zone \
  --type=A \
  --routing-policy-type=GEO \
  --routing-policy-data="us-east1=203.0.113.10;europe-west1=203.0.113.20;asia-southeast1=203.0.113.30"
```

---

## 🔹 Cost Management & Planning

### Google Cloud Pricing Calculator

**Enterprise Cost Planning:**
The Pricing Calculator is essential for budget planning, cost optimization, and architectural decision-making in enterprise environments.

**Advanced Cost Estimation Strategies:**

**Multi-Service Architecture Costing:**
```bash
# Example enterprise architecture cost components:

# Compute Engine (Web tier)
# - 10 x n1-standard-4 instances (us-central1)
# - 24/7 operation
# - Sustained use discount applied
# - Committed use discount (1-year)

# Google Kubernetes Engine (Application tier)
# - 3-node cluster (n1-standard-2)
# - Auto-scaling 3-20 nodes
# - Regional persistent disks (SSD)

# Cloud SQL (Database tier)
# - PostgreSQL db-n1-standard-8
# - High availability configuration
# - 1TB storage + automated backups

# Cloud Storage (Data tier)
# - 10TB Standard storage
# - 1TB Nearline storage (backups)
# - Data transfer costs (egress)

# Networking
# - Cloud Load Balancer
# - Cloud CDN
# - VPC peering charges
```

**Cost Optimization Techniques:**

**Committed Use Discounts:**
```bash
# Purchase compute commitments
gcloud compute commitments create my-commitment \
  --plan=12-month \
  --region=us-central1 \
  --resources=type=n1-standard-4,count=10
```

**Preemptible Instance Strategy:**
```bash
# Create preemptible instance template
gcloud compute instance-templates create preemptible-template \
  --machine-type=n1-standard-4 \
  --preemptible \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-ssd
```

**Budget Alerts and Monitoring:**
```bash
# Create budget with alerts
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Production Environment Budget" \
  --budget-amount=10000 \
  --threshold-rules-percent=50,75,90,100 \
  --notification-channels=NOTIFICATION_CHANNEL_ID
```

**Cost Attribution and Chargeback:**
```bash
# Label resources for cost attribution
gcloud compute instances create web-server \
  --labels=environment=production,team=frontend,cost-center=engineering \
  --zone=us-central1-a
```

---

## 🔹 Hybrid & Multi-Cloud Management

### Anthos Platform Architecture

**Unified Multi-Cloud Management:**
Anthos provides a consistent platform for managing applications across Google Cloud, on-premises, and other cloud providers.

**Core Anthos Components:**

**Anthos GKE (Google Kubernetes Engine):**
- Managed Kubernetes in Google Cloud
- Consistent API and tooling
- Integrated security and monitoring

**Anthos on VMware:**
- Kubernetes on existing VMware infrastructure
- Hybrid connectivity to Google Cloud
- Consistent management plane

**Anthos on Bare Metal:**
- Kubernetes on physical servers
- Edge computing capabilities
- Air-gapped environment support

**Anthos Service Mesh (ASM):**
```yaml
# ASM configuration for multi-cluster service mesh
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: control-plane
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
  components:
    pilot:
      k8s:
        env:
          - name: PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION
            value: true
```

**Multi-Cluster Management:**
```bash
# Register cluster with Anthos
gcloud container hub memberships register cluster1 \
  --gke-cluster=us-central1-a/production-cluster \
  --enable-workload-identity

# Configure Config Management
gcloud alpha container hub config-management apply \
  --membership=cluster1 \
  --config=config-management.yaml
```

```yaml
# config-management.yaml
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  git:
    syncRepo: https://github.com/company/k8s-configs
    syncBranch: main
    secretType: ssh
    policyDir: "clusters/production"
  policyController:
    enabled: true
    referentialRulesEnabled: true
    logDeniesEnabled: true
```

**Cross-Cloud Networking:**
```bash
# Create VPN connection to on-premises
gcloud compute vpn-gateways create on-prem-gateway \
  --network=anthos-network \
  --region=us-central1

gcloud compute vpn-tunnels create on-prem-tunnel \
  --peer-address=203.0.113.100 \
  --shared-secret=shared-secret-key \
  --target-vpn-gateway=on-prem-gateway \
  --region=us-central1
```

### Enterprise Hybrid Patterns

**Application Modernization Strategy:**
```yaml
# Gradual migration using Anthos
# Phase 1: Lift and shift to containers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      containers:
      - name: app
        image: gcr.io/project/legacy-app:v1
        ports:
        - containerPort: 8080

---
# Phase 2: Microservices decomposition
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: service
        image: gcr.io/project/user-service:v2
        ports:
        - containerPort: 8080
```

---

## 🔹 API Management & Integration

### Apigee API Management Platform

**Enterprise API Strategy:**
Apigee provides comprehensive API lifecycle management, essential for digital transformation and ecosystem development.

**API Proxy Configuration:**
```xml
<!-- API Proxy definition -->
<APIProxy name="products-api">
  <ProxyEndpoints>
    <ProxyEndpoint name="default">
      <HTTPProxyConnection>
        <BasePath>/v1/products</BasePath>
        <VirtualHost>secure</VirtualHost>
      </HTTPProxyConnection>
      <Flows>
        <Flow name="Get Products">
          <Condition>(proxy.pathsuffix MatchesPath "/") and (request.verb = "GET")</Condition>
          <Request>
            <Step>
              <Name>Verify-API-Key</Name>
            </Step>
            <Step>
              <Name>Quota</Name>
            </Step>
            <Step>
              <Name>Spike-Arrest</Name>
            </Step>
          </Request>
          <Response>
            <Step>
              <Name>Add-CORS</Name>
            </Step>
          </Response>
        </Flow>
      </Flows>
      <RouteRule name="default">
        <TargetEndpoint>backend</TargetEndpoint>
      </RouteRule>
    </ProxyEndpoint>
  </ProxyEndpoints>
  <TargetEndpoints>
    <TargetEndpoint name="backend">
      <HTTPTargetConnection>
        <URL>https://backend-api.example.com</URL>
      </HTTPTargetConnection>
    </TargetEndpoint>
  </TargetEndpoints>
</APIProxy>
```

**Security Policies:**
```xml
<!-- OAuth 2.0 verification policy -->
<OAuthV2 name="Verify-OAuth-Token">
  <Operation>VerifyAccessToken</Operation>
  <AccessToken ref="request.header.authorization"/>
  <Scope>read write</Scope>
</OAuthV2>

<!-- Rate limiting policy -->
<Quota name="QuotaPolicy">
  <Allow count="1000" countRef="verifyapikey.Verify-API-Key.apiproduct.developer.quota.limit"/>
  <Interval ref="verifyapikey.Verify-API-Key.apiproduct.developer.quota.interval">1</Interval>
  <TimeUnit ref="verifyapikey.Verify-API-Key.apiproduct.developer.quota.timeunit">hour</TimeUnit>
  <Identifier ref="verifyapikey.Verify-API-Key.client_id"/>
</Quota>

<!-- Spike arrest policy -->
<SpikeArrest name="Spike-Arrest">
  <Rate>100ps</Rate>
  <Identifier ref="client_ip"/>
</SpikeArrest>
```

**API Analytics and Monitoring:**
```javascript
// Custom analytics policy
var analytics = {
  "timestamp": Date.now(),
  "api_proxy": context.getVariable("apiproxy.name"),
  "client_id": context.getVariable("verifyapikey.Verify-API-Key.client_id"),
  "response_time": context.getVariable("client.received.end.timestamp") - 
                   context.getVariable("client.received.start.timestamp"),
  "status_code": context.getVariable("message.status.code"),
  "request_size": context.getVariable("request.header.content-length"),
  "response_size": context.getVariable("response.header.content-length")
};

// Send to analytics system
context.setVariable("analytics.data", JSON.stringify(analytics));
```

**Developer Portal Configuration:**
```yaml
# OpenAPI specification for developer portal
openapi: 3.0.0
info:
  title: Products API
  version: 1.0.0
  description: Enterprise product catalog API
servers:
  - url: https://api.example.com/v1
paths:
  /products:
    get:
      summary: Get products
      security:
        - ApiKeyAuth: []
      parameters:
        - name: category
          in: query
          schema:
            type: string
        - name: limit
          in: query
          schema:
            type: integer
            maximum: 100
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Product'
components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
  schemas:
    Product:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        price:
          type: number
```

---

## 🔹 Machine Learning & AI Platform

### ML Platform Architecture

**Comprehensive ML Ecosystem:**
Google Cloud provides a complete ML platform from data preparation to model deployment and monitoring.

**ML Service Categories:**

**Pre-built AI APIs:**
```python
# Vision API for image analysis
from google.cloud import vision

def analyze_image(image_path):
    client = vision.ImageAnnotatorClient()
    
    with open(image_path, 'rb') as image_file:
        content = image_file.read()
    
    image = vision.Image(content=content)
    
    # Detect objects
    objects = client.object_localization(image=image).localized_object_annotations
    
    # Detect text
    texts = client.text_detection(image=image).text_annotations
    
    # Detect faces
    faces = client.face_detection(image=image).face_annotations
    
    return {
        'objects': [obj.name for obj in objects],
        'text': texts[0].description if texts else None,
        'faces_count': len(faces)
    }

# Natural Language API for text analysis
from google.cloud import language_v1

def analyze_sentiment(text):
    client = language_v1.LanguageServiceClient()
    
    document = language_v1.Document(
        content=text,
        type_=language_v1.Document.Type.PLAIN_TEXT
    )
    
    # Analyze sentiment
    sentiment_response = client.analyze_sentiment(
        request={'document': document}
    )
    
    # Extract entities
    entities_response = client.analyze_entities(
        request={'document': document}
    )
    
    return {
        'sentiment_score': sentiment_response.document_sentiment.score,
        'sentiment_magnitude': sentiment_response.document_sentiment.magnitude,
        'entities': [entity.name for entity in entities_response.entities]
    }
```

**AutoML for Custom Models:**
```python
# AutoML Tables for structured data
from google.cloud import automl

def create_automl_model(project_id, dataset_id, model_name):
    client = automl.AutoMlClient()
    project_location = f"projects/{project_id}/locations/us-central1"
    
    # Create model
    model = automl.Model(
        display_name=model_name,
        dataset_id=dataset_id,
        tables_model_metadata=automl.TablesModelMetadata(
            train_budget_milli_node_hours=1000,
            target_column_spec_id="target_column"
        )
    )
    
    operation = client.create_model(
        parent=project_location,
        model=model
    )
    
    print(f"Training operation: {operation.operation.name}")
    return operation

# Deploy model for predictions
def deploy_model(project_id, model_id):
    client = automl.AutoMlClient()
    model_path = client.model_path(project_id, "us-central1", model_id)
    
    operation = client.deploy_model(name=model_path)
    print(f"Deploy operation: {operation.operation.name}")
    return operation
```

**Vertex AI for MLOps:**
```python
# Vertex AI pipeline for ML workflow
from google.cloud import aiplatform
from kfp.v2 import dsl
from kfp.v2.dsl import component, pipeline

@component(
    packages_to_install=["pandas", "scikit-learn"],
    base_image="python:3.8"
)
def train_model(
    dataset_path: str,
    model_path: str
) -> str:
    import pandas as pd
    from sklearn.ensemble import RandomForestClassifier
    import joblib
    
    # Load data
    df = pd.read_csv(dataset_path)
    X = df.drop('target', axis=1)
    y = df['target']
    
    # Train model
    model = RandomForestClassifier(n_estimators=100)
    model.fit(X, y)
    
    # Save model
    joblib.dump(model, model_path)
    return model_path

@component(
    packages_to_install=["scikit-learn"],
    base_image="python:3.8"
)
def evaluate_model(
    model_path: str,
    test_dataset_path: str
) -> float:
    import pandas as pd
    import joblib
    from sklearn.metrics import accuracy_score
    
    # Load model and test data
    model = joblib.load(model_path)
    test_df = pd.read_csv(test_dataset_path)
    
    X_test = test_df.drop('target', axis=1)
    y_test = test_df['target']
    
    # Evaluate
    predictions = model.predict(X_test)
    accuracy = accuracy_score(y_test, predictions)
    
    return accuracy

@pipeline(name="ml-training-pipeline")
def ml_pipeline(
    dataset_path: str = "gs://bucket/train.csv",
    test_dataset_path: str = "gs://bucket/test.csv",
    model_path: str = "gs://bucket/model.pkl"
):
    train_task = train_model(
        dataset_path=dataset_path,
        model_path=model_path
    )
    
    evaluate_task = evaluate_model(
        model_path=train_task.output,
        test_dataset_path=test_dataset_path
    )
```

---

## 🔹 Identity & Authentication Services

### Identity Platform vs Cloud IAM

**Authentication Architecture Patterns:**

**Cloud IAM for Infrastructure Access:**
```bash
# Service account for application authentication
gcloud iam service-accounts create app-service-account \
  --display-name="Application Service Account"

# Grant minimal required permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:app-service-account@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Use workload identity for GKE
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]" \
  app-service-account@PROJECT_ID.iam.gserviceaccount.com
```

**Identity Platform for End-User Authentication:**
```javascript
// Firebase Auth integration for web applications
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInWithEmailAndPassword,
  signInWithPopup,
  GoogleAuthProvider,
  FacebookAuthProvider
} from 'firebase/auth';

const firebaseConfig = {
  apiKey: "your-api-key",
  authDomain: "your-project.firebaseapp.com",
  projectId: "your-project-id"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);

// Email/password authentication
async function signInWithEmail(email, password) {
  try {
    const userCredential = await signInWithEmailAndPassword(auth, email, password);
    const user = userCredential.user;
    
    // Get ID token for backend authentication
    const idToken = await user.getIdToken();
    
    // Send token to backend
    const response = await fetch('/api/authenticate', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${idToken}`,
        'Content-Type': 'application/json'
      }
    });
    
    return response.json();
  } catch (error) {
    console.error('Authentication error:', error);
    throw error;
  }
}

// Social authentication
async function signInWithGoogle() {
  const provider = new GoogleAuthProvider();
  provider.addScope('profile');
  provider.addScope('email');
  
  try {
    const result = await signInWithPopup(auth, provider);
    const user = result.user;
    const idToken = await user.getIdToken();
    
    return { user, idToken };
  } catch (error) {
    console.error('Google sign-in error:', error);
    throw error;
  }
}
```

**Backend Token Verification:**
```python
# Python backend for token verification
from google.oauth2 import id_token
from google.auth.transport import requests
import firebase_admin
from firebase_admin import auth

# Initialize Firebase Admin SDK
firebase_admin.initialize_app()

def verify_firebase_token(id_token_string):
    """Verify Firebase ID token"""
    try:
        # Verify the token
        decoded_token = auth.verify_id_token(id_token_string)
        uid = decoded_token['uid']
        email = decoded_token.get('email')
        
        # Get additional user claims
        user_record = auth.get_user(uid)
        custom_claims = user_record.custom_claims or {}
        
        return {
            'uid': uid,
            'email': email,
            'verified': decoded_token.get('email_verified', False),
            'custom_claims': custom_claims
        }
    except Exception as e:
        print(f'Token verification failed: {e}')
        return None

def set_custom_claims(uid, claims):
    """Set custom claims for user"""
    try:
        auth.set_custom_user_claims(uid, claims)
        return True
    except Exception as e:
        print(f'Failed to set custom claims: {e}')
        return False

# Flask route example
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/authenticate', methods=['POST'])
def authenticate():
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return jsonify({'error': 'Missing or invalid authorization header'}), 401
    
    id_token = auth_header.split('Bearer ')[1]
    user_info = verify_firebase_token(id_token)
    
    if not user_info:
        return jsonify({'error': 'Invalid token'}), 401
    
    # Check user permissions
    if 'admin' in user_info.get('custom_claims', {}):
        return jsonify({
            'message': 'Admin access granted',
            'user': user_info
        })
    else:
        return jsonify({
            'message': 'User access granted',
            'user': user_info
        })
```

**SAML Integration for Enterprise SSO:**
```xml
<!-- SAML configuration for enterprise identity providers -->
<saml:Assertion xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion">
  <saml:Issuer>https://identity-provider.company.com</saml:Issuer>
  <saml:Subject>
    <saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
      user@company.com
    </saml:NameID>
  </saml:Subject>
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>user@company.com</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="groups">
      <saml:AttributeValue>developers</saml:AttributeValue>
      <saml:AttributeValue>admin</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

---

## 🔹 Event-Driven Architecture

### Event-Driven Patterns and Implementation

**Microservices Communication:**
Event-driven architecture enables loose coupling and scalability in distributed systems.

**Event Flow Example:**
```
Order Service → OrderCreated Event → Pub/Sub Topic
                     ↓
    ┌─────────────────┼─────────────────┐
    ↓                 ↓                 ↓
Billing Service   Inventory Service   Email Service
    ↓                 ↓                 ↓
OrderBilled      StockReserved      OrderConfirmation
```

**Cloud Pub/Sub Implementation:**
```python
# Publisher service
from google.cloud import pubsub_v1
import json

class OrderEventPublisher:
    def __init__(self, project_id):
        self.project_id = project_id
        self.publisher = pubsub_v1.PublisherClient()
        
    def publish_order_created(self, order_data):
        topic_path = self.publisher.topic_path(self.project_id, 'order-events')
        
        event_data = {
            'event_type': 'OrderCreated',
            'timestamp': datetime.utcnow().isoformat(),
            'order_id': order_data['order_id'],
            'customer_id': order_data['customer_id'],
            'items': order_data['items'],
            'total_amount': order_data['total_amount']
        }
        
        # Publish event
        message_data = json.dumps(event_data).encode('utf-8')
        future = self.publisher.publish(topic_path, message_data)
        
        return future.result()

# Subscriber service
class BillingEventSubscriber:
    def __init__(self, project_id, subscription_name):
        self.project_id = project_id
        self.subscription_name = subscription_name
        self.subscriber = pubsub_v1.SubscriberClient()
        
    def process_order_events(self):
        subscription_path = self.subscriber.subscription_path(
            self.project_id, self.subscription_name
        )
        
        def callback(message):
            try:
                event_data = json.loads(message.data.decode('utf-8'))
                
                if event_data['event_type'] == 'OrderCreated':
                    self.process_billing(event_data)
                    message.ack()
                else:
                    message.ack()  # Ignore other event types
                    
            except Exception as e:
                print(f'Error processing message: {e}')
                message.nack()
        
        # Start listening
        streaming_pull_future = self.subscriber.subscribe(
            subscription_path, callback=callback
        )
        
        try:
            streaming_pull_future.result()
        except KeyboardInterrupt:
            streaming_pull_future.cancel()
    
    def process_billing(self, order_data):
        # Process billing logic
        billing_result = self.charge_customer(
            order_data['customer_id'],
            order_data['total_amount']
        )
        
        if billing_result['success']:
            # Publish billing completed event
            self.publish_billing_completed(order_data['order_id'])
```

### CloudEvents Standard Implementation

**CloudEvents Format:**
```json
{
  "specversion": "1.0",
  "type": "com.company.order.created",
  "source": "https://api.company.com/orders",
  "id": "order-12345",
  "time": "2023-12-01T10:30:00Z",
  "datacontenttype": "application/json",
  "data": {
    "order_id": "12345",
    "customer_id": "cust-789",
    "items": [
      {
        "product_id": "prod-001",
        "quantity": 2,
        "price": 29.99
      }
    ],
    "total_amount": 59.98
  }
}
```

**CloudEvents Publisher:**
```python
from cloudevents.http import CloudEvent
from cloudevents.conversion import to_json
import requests

def publish_cloudevent(event_type, source, data):
    # Create CloudEvent
    event = CloudEvent({
        "type": event_type,
        "source": source,
        "id": str(uuid.uuid4()),
        "time": datetime.utcnow().isoformat() + "Z"
    }, data)
    
    # Convert to JSON
    event_json = to_json(event)
    
    # Publish to Pub/Sub
    topic_path = publisher.topic_path(project_id, 'cloudevents-topic')
    future = publisher.publish(topic_path, event_json)
    
    return future.result()
```

### Eventarc Integration

**Eventarc Trigger Configuration:**
```bash
# Create trigger for Cloud Storage events
gcloud eventarc triggers create storage-trigger \
  --destination-run-service=file-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=upload-bucket" \
  --service-account=eventarc-sa@PROJECT_ID.iam.gserviceaccount.com

# Create trigger for Audit Log events
gcloud eventarc triggers create compute-audit-trigger \
  --destination-run-service=audit-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.audit.log.v1.written" \
  --event-filters="serviceName=compute.googleapis.com" \
  --event-filters="methodName=compute.instances.insert" \
  --service-account=eventarc-sa@PROJECT_ID.iam.gserviceaccount.com
```

**Cloud Run Event Handler:**
```python
# Cloud Run service to handle Eventarc events
import functions_framework
from cloudevents.http import from_http

@functions_framework.cloud_event
def handle_storage_event(cloud_event):
    """Handle Cloud Storage events via Eventarc"""
    
    # Extract event data
    event_data = cloud_event.data
    bucket_name = event_data['bucket']
    file_name = event_data['name']
    
    print(f"Processing file: {file_name} from bucket: {bucket_name}")
    
    # Process the file
    if file_name.endswith('.csv'):
        process_csv_file(bucket_name, file_name)
    elif file_name.endswith('.json'):
        process_json_file(bucket_name, file_name)
    
    return {'status': 'processed', 'file': file_name}

def process_csv_file(bucket_name, file_name):
    """Process CSV file"""
    from google.cloud import storage
    import pandas as pd
    
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(file_name)
    
    # Download and process
    content = blob.download_as_text()
    df = pd.read_csv(StringIO(content))
    
    # Process data
    processed_data = df.groupby('category').sum()
    
    # Save results
    result_blob = bucket.blob(f"processed/{file_name}")
    result_blob.upload_from_string(processed_data.to_csv())
```

---

## 🔹 Observability & Monitoring

### OpenTelemetry Implementation

**Comprehensive Observability Strategy:**
OpenTelemetry provides vendor-neutral observability instrumentation for modern applications.

**Application Instrumentation:**
```python
# Python application with OpenTelemetry
from opentelemetry import trace, metrics
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.exporter.cloud_monitoring import CloudMonitoringMetricsExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

# Configure tracing
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Configure Cloud Trace exporter
cloud_trace_exporter = CloudTraceSpanExporter()
span_processor = BatchSpanProcessor(cloud_trace_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Configure metrics
metric_reader = PeriodicExportingMetricReader(
    CloudMonitoringMetricsExporter(),
    export_interval_millis=60000
)
metrics.set_meter_provider(MeterProvider(metric_readers=[metric_reader]))
meter = metrics.get_meter(__name__)

# Create custom metrics
request_counter = meter.create_counter(
    "http_requests_total",
    description="Total HTTP requests"
)

response_time_histogram = meter.create_histogram(
    "http_request_duration_seconds",
    description="HTTP request duration"
)

# Instrumented function
def process_order(order_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        
        start_time = time.time()
        
        try:
            # Simulate order processing
            with tracer.start_as_current_span("validate_order"):
                validate_order(order_id)
            
            with tracer.start_as_current_span("charge_payment"):
                charge_payment(order_id)
            
            with tracer.start_as_current_span("update_inventory"):
                update_inventory(order_id)
            
            # Record success metrics
            request_counter.add(1, {"status": "success", "operation": "process_order"})
            span.set_attribute("order.status", "completed")
            
        except Exception as e:
            # Record error metrics
            request_counter.add(1, {"status": "error", "operation": "process_order"})
            span.set_attribute("order.status", "failed")
            span.record_exception(e)
            raise
        
        finally:
            # Record response time
            duration = time.time() - start_time
            response_time_histogram.record(duration, {"operation": "process_order"})
```

**Distributed Tracing Across Services:**
```python
# Service A - Order Service
from opentelemetry.propagate import inject
import requests

def create_order(order_data):
    with tracer.start_as_current_span("create_order") as span:
        # Add order attributes
        span.set_attribute("order.customer_id", order_data["customer_id"])
        span.set_attribute("order.total", order_data["total"])
        
        # Save order to database
        order_id = save_order(order_data)
        span.set_attribute("order.id", order_id)
        
        # Call billing service with trace context
        headers = {}
        inject(headers)
        
        response = requests.post(
            "https://billing-service/charge",
            json={"order_id": order_id, "amount": order_data["total"]},
            headers=headers
        )
        
        if response.status_code == 200:
            span.set_attribute("billing.status", "success")
        else:
            span.set_attribute("billing.status", "failed")
            raise Exception("Billing failed")
        
        return order_id

# Service B - Billing Service
from opentelemetry.propagate import extract

def charge_customer(request):
    # Extract trace context from incoming request
    parent_context = extract(request.headers)
    
    with tracer.start_as_current_span("charge_customer", context=parent_context) as span:
        order_id = request.json["order_id"]
        amount = request.json["amount"]
        
        span.set_attribute("billing.order_id", order_id)
        span.set_attribute("billing.amount", amount)
        
        # Process payment
        payment_result = process_payment(amount)
        
        if payment_result["success"]:
            span.set_attribute("payment.transaction_id", payment_result["transaction_id"])
            return {"status": "success", "transaction_id": payment_result["transaction_id"]}
        else:
            span.set_attribute("payment.error", payment_result["error"])
            raise Exception("Payment processing failed")
```

**Custom Metrics Dashboard:**
```yaml
# Cloud Monitoring dashboard configuration
displayName: "Application Performance Dashboard"
mosaicLayout:
  tiles:
  - width: 6
    height: 4
    widget:
      title: "HTTP Request Rate"
      xyChart:
        dataSets:
        - timeSeriesQuery:
            timeSeriesFilter:
              filter: 'metric.type="custom.googleapis.com/http_requests_total"'
              aggregation:
                alignmentPeriod: "60s"
                perSeriesAligner: "ALIGN_RATE"
                crossSeriesReducer: "REDUCE_SUM"
                groupByFields: ["metric.label.status"]
  - width: 6
    height: 4
    widget:
      title: "Response Time Percentiles"
      xyChart:
        dataSets:
        - timeSeriesQuery:
            timeSeriesFilter:
              filter: 'metric.type="custom.googleapis.com/http_request_duration_seconds"'
              aggregation:
                alignmentPeriod: "60s"
                perSeriesAligner: "ALIGN_DELTA"
                crossSeriesReducer: "REDUCE_PERCENTILE_95"
```

---

## 🔹 Service Discovery & Networking

### Service Directory Implementation

**Centralized Service Discovery:**
Service Directory provides a unified service registry for multi-cloud and hybrid environments.

**Service Registration:**
```bash
# Create namespace
gcloud service-directory namespaces create production \
  --location=us-central1

# Create service
gcloud service-directory services create user-service \
  --namespace=production \
  --location=us-central1

# Register service endpoint
gcloud service-directory endpoints create user-service-1 \
  --service=user-service \
  --namespace=production \
  --location=us-central1 \
  --address=10.0.1.100 \
  --port=8080 \
  --metadata=version=v1.2.0,environment=production
```

**DNS-based Service Discovery:**
```bash
# Configure DNS zone for service discovery
gcloud dns managed-zones create service-discovery \
  --description="Service discovery DNS zone" \
  --dns-name=services.internal. \
  --visibility=private \
  --networks=projects/PROJECT_ID/global/networks/vpc-network

# Create DNS record for service
gcloud dns record-sets transaction start --zone=service-discovery
gcloud dns record-sets transaction add \
  --name=user-service.services.internal. \
  --ttl=30 \
  --type=A \
  --zone=service-discovery \
  "10.0.1.100" "10.0.1.101"
gcloud dns record-sets transaction execute --zone=service-discovery
```

**Application Integration:**
```python
# Python client for Service Directory
from google.cloud import servicedirectory_v1

class ServiceDiscoveryClient:
    def __init__(self, project_id, location, namespace):
        self.client = servicedirectory_v1.RegistrationServiceClient()
        self.project_id = project_id
        self.location = location
        self.namespace = namespace
        self.namespace_path = self.client.namespace_path(
            project_id, location, namespace
        )
    
    def register_service(self, service_name, endpoint_address, endpoint_port, metadata=None):
        """Register a service endpoint"""
        service_path = self.client.service_path(
            self.project_id, self.location, self.namespace, service_name
        )
        
        # Create service if it doesn't exist
        try:
            service = servicedirectory_v1.Service(name=service_name)
            self.client.create_service(
                parent=self.namespace_path,
                service_id=service_name,
                service=service
            )
        except Exception:
            pass  # Service already exists
        
        # Create endpoint
        endpoint = servicedirectory_v1.Endpoint(
            address=endpoint_address,
            port=endpoint_port,
            metadata=metadata or {}
        )
        
        endpoint_id = f"{service_name}-{endpoint_address}-{endpoint_port}"
        
        self.client.create_endpoint(
            parent=service_path,
            endpoint_id=endpoint_id,
            endpoint=endpoint
        )
    
    def discover_service(self, service_name):
        """Discover service endpoints"""
        service_path = self.client.service_path(
            self.project_id, self.location, self.namespace, service_name
        )
        
        endpoints = self.client.list_endpoints(parent=service_path)
        
        return [
            {
                'address': endpoint.address,
                'port': endpoint.port,
                'metadata': dict(endpoint.metadata)
            }
            for endpoint in endpoints
        ]

# Usage in application
service_discovery = ServiceDiscoveryClient(
    project_id="my-project",
    location="us-central1",
    namespace="production"
)

# Register current service
service_discovery.register_service(
    service_name="api-service",
    endpoint_address="10.0.1.50",
    endpoint_port=8080,
    metadata={
        "version": "v2.1.0",
        "environment": "production",
        "health_check": "/health"
    }
)

# Discover dependency services
user_service_endpoints = service_discovery.discover_service("user-service")
for endpoint in user_service_endpoints:
    print(f"User service available at {endpoint['address']}:{endpoint['port']}")
```

**Load Balancing with Service Discovery:**
```python
import random
import requests
from typing import List, Dict

class ServiceClient:
    def __init__(self, service_discovery_client, service_name):
        self.service_discovery = service_discovery_client
        self.service_name = service_name
        self.endpoints_cache = []
        self.cache_ttl = 30  # seconds
        self.last_refresh = 0
    
    def get_healthy_endpoints(self) -> List[Dict]:
        """Get list of healthy service endpoints"""
        current_time = time.time()
        
        # Refresh cache if needed
        if current_time - self.last_refresh > self.cache_ttl:
            self.refresh_endpoints()
        
        # Filter healthy endpoints
        healthy_endpoints = []
        for endpoint in self.endpoints_cache:
            if self.health_check(endpoint):
                healthy_endpoints.append(endpoint)
        
        return healthy_endpoints
    
    def refresh_endpoints(self):
        """Refresh endpoints from service discovery"""
        try:
            self.endpoints_cache = self.service_discovery.discover_service(
                self.service_name
            )
            self.last_refresh = time.time()
        except Exception as e:
            print(f"Failed to refresh endpoints: {e}")
    
    def health_check(self, endpoint: Dict) -> bool:
        """Check if endpoint is healthy"""
        try:
            health_path = endpoint['metadata'].get('health_check', '/health')
            url = f"http://{endpoint['address']}:{endpoint['port']}{health_path}"
            
            response = requests.get(url, timeout=5)
            return response.status_code == 200
        except Exception:
            return False
    
    def make_request(self, path: str, method: str = 'GET', **kwargs):
        """Make request to service with load balancing"""
        healthy_endpoints = self.get_healthy_endpoints()
        
        if not healthy_endpoints:
            raise Exception(f"No healthy endpoints for service {self.service_name}")
        
        # Simple round-robin load balancing
        endpoint = random.choice(healthy_endpoints)
        url = f"http://{endpoint['address']}:{endpoint['port']}{path}"
        
        return requests.request(method, url, **kwargs)

# Usage
user_service_client = ServiceClient(service_discovery, "user-service")
response = user_service_client.make_request("/api/users/123")
```

---

## 🔹 Modern CLI & Storage Management

### gcloud storage CLI Evolution

**Next-Generation Storage CLI:**
The new `gcloud storage` command provides significant performance improvements and simplified syntax over `gsutil`.

**Performance Improvements:**
- Up to 94% faster data transfers
- Automatic parallelism and optimization
- Improved error handling and retry logic
- Better progress reporting

**Migration from gsutil:**
```bash
# Old gsutil commands
gsutil cp file.txt gs://bucket/
gsutil -m cp -r directory/ gs://bucket/
gsutil rsync -r -d local-dir gs://bucket/remote-dir

# New gcloud storage commands
gcloud storage cp file.txt gs://bucket/
gcloud storage cp -r directory/ gs://bucket/
gcloud storage rsync local-dir gs://bucket/remote-dir
```

**Advanced Storage Operations:**
```bash
# Bucket lifecycle management
gcloud storage buckets create gs://enterprise-data \
  --location=us-central1 \
  --storage-class=STANDARD \
  --uniform-bucket-level-access

# Configure lifecycle policy
cat > lifecycle.json << EOF
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
      "condition": {"age": 90}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365}
    }
  ]
}
EOF

gcloud storage buckets update gs://enterprise-data --lifecycle-file=lifecycle.json

# Parallel uploads with resumable transfers
gcloud storage cp large-file.zip gs://bucket/ \
  --parallel-composite-upload-threshold=150M

# Sync with deletion and exclusion patterns
gcloud storage rsync source-dir gs://bucket/target-dir \
  --delete-unmatched-destination-objects \
  --exclude="*.tmp" \
  --exclude="*.log"
```

**Enterprise Storage Patterns:**
```bash
# Multi-regional bucket for global access
gcloud storage buckets create gs://global-assets \
  --location=us \
  --storage-class=STANDARD \
  --public-access-prevention

# Configure CORS for web applications
cat > cors.json << EOF
[
  {
    "origin": ["https://app.example.com"],
    "method": ["GET", "POST", "PUT"],
    "responseHeader": ["Content-Type", "x-custom-header"],
    "maxAgeSeconds": 3600
  }
]
EOF

gcloud storage buckets update gs://global-assets --cors-file=cors.json

# Set up notifications for file processing
gcloud storage buckets notifications create gs://upload-bucket \
  --topic=projects/PROJECT_ID/topics/file-processing \
  --event-types=OBJECT_FINALIZE \
  --payload-format=json
```

**Automated Backup Strategy:**
```bash
#!/bin/bash
# Enterprise backup script using gcloud storage

BACKUP_SOURCE="/data/production"
BACKUP_BUCKET="gs://enterprise-backups"
DATE=$(date +%Y-%m-%d)
BACKUP_PATH="$BACKUP_BUCKET/daily/$DATE"

# Create compressed backup
tar -czf - "$BACKUP_SOURCE" | \
gcloud storage cp - "$BACKUP_PATH/production-backup.tar.gz"

# Verify backup
gcloud storage ls -l "$BACKUP_PATH/"

# Clean up old backups (keep 30 days)
CUTOFF_DATE=$(date -d '30 days ago' +%Y-%m-%d)
gcloud storage rm -r "$BACKUP_BUCKET/daily/$CUTOFF_DATE" 2>/dev/null || true

# Send notification
echo "Backup completed: $BACKUP_PATH" | \
gcloud pubsub topics publish backup-notifications --message=-
```

**Storage Analytics and Monitoring:**
```python
# Python script for storage analytics
from google.cloud import storage
from google.cloud import monitoring_v3
import datetime

def analyze_storage_usage(project_id, bucket_name):
    """Analyze storage usage patterns"""
    
    storage_client = storage.Client(project=project_id)
    monitoring_client = monitoring_v3.MetricServiceClient()
    
    bucket = storage_client.bucket(bucket_name)
    
    # Get bucket metrics
    project_name = f"projects/{project_id}"
    
    # Query storage metrics
    interval = monitoring_v3.TimeInterval({
        "end_time": {"seconds": int(datetime.datetime.now().timestamp())},
        "start_time": {"seconds": int((datetime.datetime.now() - datetime.timedelta(days=7)).timestamp())}
    })
    
    # Storage bytes metric
    results = monitoring_client.list_time_series(
        request={
            "name": project_name,
            "filter": f'metric.type="storage.googleapis.com/storage/total_bytes" AND resource.label.bucket_name="{bucket_name}"',
            "interval": interval,
            "view": monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL
        }
    )
    
    # Analyze results
    for result in results:
        for point in result.points:
            timestamp = point.interval.end_time
            value = point.value.int64_value
            print(f"Storage usage at {timestamp}: {value / (1024**3):.2f} GB")
    
    # Get object count and size distribution
    total_objects = 0
    total_size = 0
    size_distribution = {"small": 0, "medium": 0, "large": 0}
    
    for blob in bucket.list_blobs():
        total_objects += 1
        total_size += blob.size
        
        if blob.size < 1024 * 1024:  # < 1MB
            size_distribution["small"] += 1
        elif blob.size < 100 * 1024 * 1024:  # < 100MB
            size_distribution["medium"] += 1
        else:
            size_distribution["large"] += 1
    
    return {
        "total_objects": total_objects,
        "total_size_gb": total_size / (1024**3),
        "size_distribution": size_distribution
    }

# Usage
analytics = analyze_storage_usage("my-project", "enterprise-data")
print(f"Total objects: {analytics['total_objects']}")
print(f"Total size: {analytics['total_size_gb']:.2f} GB")
print(f"Size distribution: {analytics['size_distribution']}")
```

This comprehensive guide covers the essential "other" GCP services that senior DevOps engineers need to understand for building enterprise-scale cloud architectures. Each service is presented with practical implementation examples, enterprise patterns, and integration strategies that reflect real-world usage scenarios.

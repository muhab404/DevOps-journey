# ☁️ Google Cloud Storage Complete Guide
*DevOps-Focused Reference*

## Table of Contents
1. [Introduction to Cloud Storage](#introduction-to-cloud-storage)
2. [Data Management in Cloud Storage](#data-management-in-cloud-storage)
3. [Security in Cloud Storage](#security-in-cloud-storage)
4. [Data Transfer Options](#data-transfer-options)
5. [Best Practices & Tools](#best-practices--tools)
6. [DevOps Integration](#devops-integration)
7. [Monitoring & Cost Optimization](#monitoring--cost-optimization)

---

## 🔹 Introduction to Cloud Storage

### What is Cloud Storage?

**Google Cloud Storage** is a unified object storage service that offers industry-leading scalability, data availability, and security. It's designed for storing and accessing unstructured data at any scale.

**Key Characteristics:**
- **Object storage** - Files stored as objects in buckets
- **Globally distributed** - Multi-region and dual-region options
- **Highly durable** - 99.999999999% (11 9's) durability
- **Scalable** - Exabyte scale with no limits
- **RESTful API** - HTTP-based access from anywhere

### Buckets and Objects (Core Concepts)

**Bucket Attributes:**
```yaml
Bucket Properties:
  name: globally-unique-bucket-name
  location: us-central1 | us | eu | asia
  storage_class: STANDARD | NEARLINE | COLDLINE | ARCHIVE
  versioning: enabled | suspended
  lifecycle_rules: [array of lifecycle rules]
  cors: [array of CORS rules]
  website: {main_page_suffix, not_found_page}
  labels: {key: value pairs}
  retention_policy: {retention_period, locked}
  iam_configuration:
    uniform_bucket_level_access: enabled | disabled
    public_access_prevention: enforced | inherited
  encryption:
    default_kms_key_name: projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY
  logging:
    log_bucket: bucket-name
    log_object_prefix: access-logs/
  notification_configs: [array of Pub/Sub notifications]
  requester_pays: enabled | disabled
```

**Object Attributes:**
```yaml
Object Properties:
  name: object-path/filename.ext
  bucket: bucket-name
  generation: 1234567890123456  # Version identifier
  metageneration: 1  # Metadata version
  content_type: application/json | text/plain | image/jpeg
  content_language: en | es | fr
  content_encoding: gzip | deflate
  content_disposition: attachment; filename="file.txt"
  cache_control: public, max-age=3600
  size: 1048576  # Size in bytes
  md5_hash: base64-encoded-md5
  crc32c: base64-encoded-crc32c
  etag: "generation/metageneration"
  time_created: 2024-01-15T10:30:00Z
  updated: 2024-01-15T10:30:00Z
  time_deleted: null | timestamp
  temporary_hold: true | false
  event_based_hold: true | false
  retention_expiration_time: timestamp
  storage_class: STANDARD | NEARLINE | COLDLINE | ARCHIVE
  kms_key_name: projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY
  customer_encryption:
    encryption_algorithm: AES256
    key_sha256: base64-encoded-key-hash
  metadata: {custom-key: custom-value}
  acl: [array of access control entries]
```

**Detailed Bucket Creation:**
```bash
# Create bucket with all options
gsutil mb \
  -p my-project-id \
  -c STANDARD \
  -l us-central1 \
  -b on \
  --retention 30d \
  --pap enforced \
  gs://my-detailed-bucket

# Bucket creation options explained:
# -p: Project ID
# -c: Storage class (STANDARD, NEARLINE, COLDLINE, ARCHIVE)
# -l: Location (region, dual-region, or multi-region)
# -b: Enable uniform bucket-level access (on/off)
# --retention: Retention period (e.g., 30d, 1y)
# --pap: Public access prevention (enforced/inherited)
```

**Advanced Object Operations:**
```bash
# Upload with metadata and options
gsutil -h "Content-Type:application/json" \
       -h "Cache-Control:public, max-age=3600" \
       -h "x-goog-meta-department:engineering" \
       -h "x-goog-meta-version:1.0" \
       cp -Z file.json gs://bucket/path/

# Upload options explained:
# -h: Set headers (Content-Type, Cache-Control, custom metadata)
# -Z: Enable gzip compression during upload
# -n: No-clobber (don't overwrite existing objects)
# -r: Recursive (for directories)
# -m: Parallel/multi-threaded operations

# Advanced copy with all options
gsutil -m cp -r -Z -n \
       -j json,txt,html \
       -x ".*\.tmp$" \
       local-dir/ gs://bucket/remote-dir/

# Copy options explained:
# -j: File extensions to compress
# -x: Exclude pattern (regex)
# -A: Preserve ACLs
# -P: Preserve POSIX attributes
```

### Cloud Storage Classes

**Storage Class Comparison:**

| Class | Availability SLA | Durability | Min Storage Duration | Retrieval Cost | Use Case |
|-------|------------------|------------|---------------------|----------------|----------|
| **Standard** | 99.95% (multi-region)<br>99.9% (region) | 99.999999999% | None | None | Frequently accessed data |
| **Nearline** | 99.95% (multi-region)<br>99.9% (region) | 99.999999999% | 30 days | $0.01/GB | Monthly access |
| **Coldline** | 99.95% (multi-region)<br>99.9% (region) | 99.999999999% | 90 days | $0.02/GB | Quarterly access |
| **Archive** | 99.95% (multi-region)<br>99.9% (region) | 99.999999999% | 365 days | $0.05/GB | Annual access |

**DevOps Decision Matrix:**
```yaml
Standard Storage:
  - Active website content
  - Application data
  - Frequently accessed logs
  - Real-time analytics data

Nearline Storage:
  - Monthly reports
  - Backup data accessed monthly
  - Content for seasonal campaigns

Coldline Storage:
  - Quarterly compliance data
  - Disaster recovery backups
  - Long-term project archives

Archive Storage:
  - Legal compliance data
  - Historical records
  - Long-term backup retention
```

### SLAs for Different Storage Classes

**Availability SLAs:**
```bash
# Multi-regional buckets
Standard, Nearline, Coldline, Archive: 99.95%

# Regional buckets  
Standard, Nearline, Coldline, Archive: 99.9%

# Dual-regional buckets
Standard, Nearline, Coldline, Archive: 99.95%
```

**Detailed Performance Characteristics:**
```yaml
Request Rate Limits:
  GET requests:
    - Standard: 5,000 requests/second per prefix
    - Nearline: 1,000 requests/second per prefix  
    - Coldline: 1,000 requests/second per prefix
    - Archive: 1,000 requests/second per prefix
  
  PUT/POST/DELETE requests:
    - All classes: 1,000 requests/second per prefix
  
  LIST requests:
    - All classes: 100 requests/second per bucket
  
  Metadata operations:
    - GET/PUT metadata: 1,000 requests/second per prefix

Throughput Limits:
  Single-stream performance:
    - Upload: ~1 Gbps per stream
    - Download: ~2 Gbps per stream
  
  Parallel operations:
    - Upload: Multiple Gbps (scales with parallelism)
    - Download: Multiple Gbps (scales with parallelism)
    - Composite uploads: Up to 32 parallel streams
  
  Network egress:
    - Same region: No bandwidth limits
    - Cross-region: Network capacity dependent
    - Internet egress: 50 Gbps per project (can be increased)

Latency Characteristics:
  First-byte latency:
    - Standard: 5-10 milliseconds (same region)
    - Standard: 50-100 milliseconds (cross-region)
    - Nearline: 5-10 milliseconds (same region)
    - Coldline: 5-10 milliseconds (same region)
    - Archive: 5-10 milliseconds (same region, after first access)
  
  Time to first byte (TTFB):
    - Small objects (<1MB): <100ms
    - Large objects (>1GB): <1s
  
  Archive restoration:
    - First access: Up to 12 hours (rare)
    - Subsequent access: Milliseconds

Size Limits:
  Object size:
    - Maximum: 5 TiB per object
    - Minimum: 0 bytes (empty objects allowed)
  
  Bucket limits:
    - Objects per bucket: Unlimited
    - Bucket name length: 3-63 characters
    - Total buckets per project: 1000 (default, can be increased)
  
  Upload limits:
    - Simple upload: 5 GiB
    - Multipart upload: 5 TiB
    - Resumable upload: 5 TiB
    - Streaming upload: 5 TiB

Consistency Model:
  Read-after-write consistency:
    - New objects: Immediate consistency
    - Object updates: Immediate consistency
    - Object deletions: Immediate consistency
  
  List consistency:
    - New objects: Eventually consistent
    - Deleted objects: Eventually consistent
    - Typical propagation: <1 second

Availability SLAs (Monthly):
  Multi-regional buckets:
    - Standard/Nearline/Coldline/Archive: 99.95%
  
  Regional buckets:
    - Standard/Nearline/Coldline/Archive: 99.9%
  
  Dual-regional buckets:
    - Standard/Nearline/Coldline/Archive: 99.95%

Durability:
  All storage classes: 99.999999999% (11 9's)
  
  Replication:
    - Regional: 3+ replicas within region
    - Multi-regional: Replicas across regions
    - Dual-regional: Replicas in 2 specific regions
```

---

## 🔹 Data Management in Cloud Storage

### Object Versioning

**Enable Versioning:**
```bash
# Enable versioning on bucket
gsutil versioning set on gs://my-bucket

# Check versioning status
gsutil versioning get gs://my-bucket
```

**Version Management:**
```bash
# List all versions of an object
gsutil ls -a gs://my-bucket/file.txt

# Download specific version
gsutil cp gs://my-bucket/file.txt#1234567890123456 ./old-version.txt

# Delete specific version
gsutil rm gs://my-bucket/file.txt#1234567890123456

# Delete all versions
gsutil rm -a gs://my-bucket/file.txt
```

**Versioning Best Practices:**
```python
#!/usr/bin/env python3
# version-management.py

from google.cloud import storage
import datetime

def manage_object_versions(bucket_name, object_name, keep_versions=5):
    """Keep only the latest N versions of an object."""
    
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    
    # List all versions
    versions = list(bucket.list_blobs(prefix=object_name, versions=True))
    
    # Sort by time created (newest first)
    versions.sort(key=lambda x: x.time_created, reverse=True)
    
    # Keep only the latest versions
    versions_to_delete = versions[keep_versions:]
    
    for version in versions_to_delete:
        print(f"Deleting version: {version.name}#{version.generation}")
        version.delete()

# Automated cleanup script
if __name__ == "__main__":
    manage_object_versions("my-bucket", "important-file.txt", keep_versions=3)
```

### Lifecycle Management

**Basic Lifecycle Rules:**
```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {
          "type": "SetStorageClass",
          "storageClass": "NEARLINE"
        },
        "condition": {
          "age": 30,
          "matchesStorageClass": ["STANDARD"]
        }
      },
      {
        "action": {
          "type": "SetStorageClass", 
          "storageClass": "COLDLINE"
        },
        "condition": {
          "age": 90,
          "matchesStorageClass": ["NEARLINE"]
        }
      },
      {
        "action": {
          "type": "Delete"
        },
        "condition": {
          "age": 2555,
          "matchesStorageClass": ["ARCHIVE"]
        }
      }
    ]
  }
}
```

**Complete Lifecycle Configuration Options:**
```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {
          "type": "SetStorageClass",
          "storageClass": "NEARLINE"
        },
        "condition": {
          "age": 30,
          "createdBefore": "2024-01-01",
          "customTimeBefore": "2024-01-01",
          "daysSinceCustomTime": 30,
          "daysSinceNoncurrentTime": 30,
          "isLive": true,
          "matchesPrefix": ["logs/", "backups/"],
          "matchesSuffix": [".log", ".bak"],
          "matchesStorageClass": ["STANDARD"],
          "numNewerVersions": 5
        }
      },
      {
        "action": {
          "type": "Delete"
        },
        "condition": {
          "age": 365,
          "matchesPrefix": ["temp/", "cache/"],
          "matchesStorageClass": ["COLDLINE", "ARCHIVE"]
        }
      },
      {
        "action": {
          "type": "AbortIncompleteMultipartUpload"
        },
        "condition": {
          "age": 7
        }
      }
    ]
  }
}
```

**Lifecycle Condition Options Explained:**
```yaml
Condition Fields:
  age: Number of days since object creation
  createdBefore: Date in YYYY-MM-DD format
  customTimeBefore: Date in YYYY-MM-DD format (custom time metadata)
  daysSinceCustomTime: Days since custom time was set
  daysSinceNoncurrentTime: Days since object became non-current
  isLive: true (current version) | false (non-current version)
  matchesPrefix: Array of object name prefixes
  matchesSuffix: Array of object name suffixes
  matchesStorageClass: Array of storage classes to match
  numNewerVersions: Number of newer versions (for versioned objects)
  
Action Types:
  SetStorageClass: Change storage class
    - storageClass: STANDARD | NEARLINE | COLDLINE | ARCHIVE
  Delete: Delete object
  AbortIncompleteMultipartUpload: Clean up incomplete uploads
```

**Advanced Lifecycle Examples:**
```bash
# Apply lifecycle policy with validation
gsutil lifecycle set lifecycle.json gs://my-bucket

# Get current lifecycle policy
gsutil lifecycle get gs://my-bucket

# Remove lifecycle policy
echo '{}' | gsutil lifecycle set /dev/stdin gs://my-bucket

# Validate lifecycle configuration
gsutil lifecycle get gs://my-bucket | jq '.lifecycle.rule[] | select(.condition.age > 365)'
```

**DevOps Lifecycle Automation:**
```python
#!/usr/bin/env python3
# lifecycle-automation.py

from google.cloud import storage
import json

class LifecycleManager:
    def __init__(self, project_id):
        self.client = storage.Client(project=project_id)
    
    def create_tiered_lifecycle(self, bucket_name, rules_config):
        """Create automated data tiering lifecycle."""
        
        bucket = self.client.bucket(bucket_name)
        
        lifecycle_rules = []
        
        for rule in rules_config:
            lifecycle_rule = {
                "action": {
                    "type": rule["action"],
                    "storageClass": rule.get("storage_class")
                },
                "condition": {
                    "age": rule["age_days"]
                }
            }
            
            # Add optional conditions
            if "prefix" in rule:
                lifecycle_rule["condition"]["matchesPrefix"] = rule["prefix"]
            
            if "storage_classes" in rule:
                lifecycle_rule["condition"]["matchesStorageClass"] = rule["storage_classes"]
            
            lifecycle_rules.append(lifecycle_rule)
        
        # Apply lifecycle policy
        bucket.lifecycle_rules = lifecycle_rules
        bucket.patch()
        
        print(f"Applied lifecycle policy to {bucket_name}")

# Usage example
lifecycle_config = [
    {
        "action": "SetStorageClass",
        "storage_class": "NEARLINE", 
        "age_days": 30,
        "storage_classes": ["STANDARD"]
    },
    {
        "action": "SetStorageClass",
        "storage_class": "COLDLINE",
        "age_days": 90, 
        "storage_classes": ["NEARLINE"]
    },
    {
        "action": "Delete",
        "age_days": 2555,
        "prefix": ["logs/", "temp/"]
    }
]

manager = LifecycleManager("my-project")
manager.create_tiered_lifecycle("my-bucket", lifecycle_config)
```

### Cloud Storage Metadata

**System Metadata (Complete List):**
```bash
# View detailed object metadata
gsutil stat gs://my-bucket/file.txt

# Complete system metadata fields:
Content-Type: application/json
Content-Length: 1048576
Content-Language: en
Content-Encoding: gzip
Content-Disposition: attachment; filename="data.json"
Cache-Control: public, max-age=3600
ETag: "CKih3ZjF8eYCEAE="
Generation: 1234567890123456
Metageneration: 1
Time-Created: Mon, 15 Jan 2024 10:30:00 GMT
Time-Updated: Mon, 15 Jan 2024 10:30:00 GMT
Time-Storage-Class-Updated: Mon, 15 Jan 2024 10:30:00 GMT
Storage-Class: STANDARD
Temporary-Hold: False
Event-Based-Hold: False
Retention-Expiration-Time: None
KMS-Key: projects/PROJECT/locations/global/keyRings/RING/cryptoKeys/KEY
Hash (crc32c): AAAAAA==
Hash (md5): 1B2M2Y8AsgTpgAmY7PhCfg==
Component-Count: 1
Custom-Metadata:
  department: engineering
  project: web-app
  version: 1.0
```

**Metadata Management Options:**
```bash
# Set multiple metadata headers
gsutil setmeta \
  -h "Content-Type:application/json" \
  -h "Content-Language:en" \
  -h "Content-Encoding:gzip" \
  -h "Cache-Control:public, max-age=86400" \
  -h "Content-Disposition:attachment; filename=data.json" \
  -h "x-goog-meta-department:engineering" \
  -h "x-goog-meta-project:web-app" \
  -h "x-goog-meta-environment:production" \
  -h "x-goog-meta-backup-policy:daily" \
  gs://my-bucket/file.json

# Remove metadata
gsutil setmeta -h "x-goog-meta-old-field:" gs://my-bucket/file.json

# Copy metadata from one object to another
gsutil cp -p gs://source-bucket/file.txt gs://dest-bucket/file.txt
# -p preserves all metadata and ACLsn
# - Time-Created
# - Updated
# - CRC32C checksum
# - MD5 hash
```

**Custom Metadata:**
```bash
# Set custom metadata
gsutil setmeta -h "x-goog-meta-department:engineering" \
               -h "x-goog-meta-project:web-app" \
               gs://my-bucket/file.txt

# Set content type
gsutil setmeta -h "Content-Type:application/json" gs://my-bucket/data.json

# Set cache control
gsutil setmeta -h "Cache-Control:public, max-age=3600" gs://my-bucket/image.jpg
```

**Metadata Management Script:**
```python
#!/usr/bin/env python3
# metadata-manager.py

from google.cloud import storage

def set_bulk_metadata(bucket_name, prefix, metadata):
    """Set metadata for multiple objects."""
    
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    
    # List objects with prefix
    blobs = bucket.list_blobs(prefix=prefix)
    
    for blob in blobs:
        # Update metadata
        blob.metadata = {**blob.metadata or {}, **metadata}
        blob.patch()
        
        print(f"Updated metadata for {blob.name}")

def organize_by_metadata(bucket_name):
    """Organize objects based on metadata."""
    
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    
    for blob in bucket.list_blobs():
        if blob.metadata:
            department = blob.metadata.get('department')
            if department:
                # Move to department-specific prefix
                new_name = f"{department}/{blob.name}"
                bucket.copy_blob(blob, bucket, new_name)
                blob.delete()
                print(f"Moved {blob.name} to {new_name}")

# Usage
metadata = {
    "environment": "production",
    "backup-policy": "daily",
    "retention": "90-days"
}

set_bulk_metadata("my-bucket", "backups/", metadata)
```

### Bucket Lock (Compliance)

**Retention Policy:**
```bash
# Set retention policy (cannot be reduced)
gsutil retention set 30d gs://compliance-bucket

# Get retention policy
gsutil retention get gs://compliance-bucket

# Lock retention policy (irreversible)
gsutil retention lock gs://compliance-bucket
```

**Legal Hold:**
```bash
# Set legal hold on object
gsutil retention leghold set gs://compliance-bucket/legal-document.pdf

# Release legal hold
gsutil retention leghold release gs://compliance-bucket/legal-document.pdf

# Check legal hold status
gsutil ls -L gs://compliance-bucket/legal-document.pdf
```

**Compliance Automation:**
```python
#!/usr/bin/env python3
# compliance-manager.py

from google.cloud import storage
import datetime

class ComplianceManager:
    def __init__(self, project_id):
        self.client = storage.Client(project=project_id)
    
    def setup_compliance_bucket(self, bucket_name, retention_days):
        """Setup bucket with compliance features."""
        
        bucket = self.client.bucket(bucket_name)
        
        # Set retention policy
        bucket.retention_period = retention_days * 24 * 3600  # Convert to seconds
        bucket.patch()
        
        # Enable uniform bucket-level access
        bucket.iam_configuration.uniform_bucket_level_access_enabled = True
        bucket.patch()
        
        print(f"Configured compliance bucket: {bucket_name}")
        print(f"Retention period: {retention_days} days")
    
    def audit_compliance_status(self, bucket_name):
        """Audit compliance status of bucket."""
        
        bucket = self.client.bucket(bucket_name)
        
        audit_report = {
            "bucket": bucket_name,
            "retention_policy": {
                "enabled": bucket.retention_period is not None,
                "period_days": bucket.retention_period // (24 * 3600) if bucket.retention_period else 0,
                "locked": bucket.retention_policy_locked
            },
            "uniform_access": bucket.iam_configuration.uniform_bucket_level_access_enabled,
            "objects_with_holds": []
        }
        
        # Check for objects with legal holds
        for blob in bucket.list_blobs():
            if blob.temporary_hold or blob.event_based_hold:
                audit_report["objects_with_holds"].append({
                    "name": blob.name,
                    "temporary_hold": blob.temporary_hold,
                    "event_based_hold": blob.event_based_hold
                })
        
        return audit_report

# Usage
compliance = ComplianceManager("my-project")
compliance.setup_compliance_bucket("legal-documents", retention_days=2555)  # 7 years
audit = compliance.audit_compliance_status("legal-documents")
print(json.dumps(audit, indent=2))
```

---

## 🔹 Security in Cloud Storage

### Encrypting Data with Cloud KMS

**Customer-Managed Encryption Keys (CMEK):**
```bash
# Create KMS key
gcloud kms keyrings create storage-keyring --location=global
gcloud kms keys create storage-key \
    --keyring=storage-keyring \
    --location=global \
    --purpose=encryption

# Create bucket with CMEK
gsutil mb -p my-project \
    -c STANDARD \
    -l us-central1 \
    -k projects/my-project/locations/global/keyRings/storage-keyring/cryptoKeys/storage-key \
    gs://encrypted-bucket
```

**Upload with Encryption:**
```bash
# Upload with customer-supplied encryption key (CSEK)
gsutil -o "GSUtil:encryption_key=YOUR_BASE64_KEY" cp file.txt gs://my-bucket/

# Upload with KMS key
gsutil -o "GSUtil:kms_key=projects/my-project/locations/global/keyRings/storage-keyring/cryptoKeys/storage-key" \
    cp file.txt gs://encrypted-bucket/
```

**Encryption Management Script:**
```python
#!/usr/bin/env python3
# encryption-manager.py

from google.cloud import storage, kms
import base64
import os

class EncryptionManager:
    def __init__(self, project_id):
        self.storage_client = storage.Client(project=project_id)
        self.kms_client = kms.KeyManagementServiceClient()
        self.project_id = project_id
    
    def create_encrypted_bucket(self, bucket_name, kms_key_name):
        """Create bucket with KMS encryption."""
        
        bucket = self.storage_client.bucket(bucket_name)
        bucket.default_kms_key_name = kms_key_name
        bucket = self.storage_client.create_bucket(bucket)
        
        print(f"Created encrypted bucket: {bucket_name}")
        print(f"KMS key: {kms_key_name}")
        
        return bucket
    
    def upload_with_csek(self, bucket_name, source_file, destination_name, encryption_key):
        """Upload file with customer-supplied encryption key."""
        
        bucket = self.storage_client.bucket(bucket_name)
        blob = bucket.blob(destination_name)
        
        # Set encryption key
        blob.encryption_key = base64.b64decode(encryption_key)
        
        # Upload file
        blob.upload_from_filename(source_file)
        
        print(f"Uploaded {source_file} with CSEK encryption")
    
    def rotate_kms_key(self, key_name):
        """Rotate KMS key and re-encrypt objects."""
        
        # Create new key version
        response = self.kms_client.create_crypto_key_version(
            request={"parent": key_name, "crypto_key_version": {}}
        )
        
        print(f"Created new key version: {response.name}")
        
        # Re-encrypt existing objects (simplified example)
        # In practice, you'd need to copy objects to trigger re-encryption
        
    def audit_encryption_status(self, bucket_name):
        """Audit encryption status of bucket and objects."""
        
        bucket = self.storage_client.bucket(bucket_name)
        
        audit_report = {
            "bucket": bucket_name,
            "default_kms_key": bucket.default_kms_key_name,
            "objects": []
        }
        
        for blob in bucket.list_blobs():
            object_info = {
                "name": blob.name,
                "kms_key": blob.kms_key_name,
                "customer_encryption": blob.customer_encryption is not None,
                "encryption_algorithm": getattr(blob.customer_encryption, 'encryption_algorithm', None)
            }
            audit_report["objects"].append(object_info)
        
        return audit_report

# Usage
encryption = EncryptionManager("my-project")

# Create KMS-encrypted bucket
kms_key = "projects/my-project/locations/global/keyRings/storage-keyring/cryptoKeys/storage-key"
bucket = encryption.create_encrypted_bucket("secure-bucket", kms_key)

# Audit encryption
audit = encryption.audit_encryption_status("secure-bucket")
```

### Meeting Compliance Requirements

**Access Control Best Practices:**
```bash
# Enable uniform bucket-level access
gsutil uniformbucketlevelaccess set on gs://compliance-bucket

# Set IAM policies
gsutil iam ch user:auditor@company.com:objectViewer gs://compliance-bucket
gsutil iam ch group:compliance-team@company.com:objectAdmin gs://compliance-bucket

# Remove public access
gsutil iam ch -d allUsers:objectViewer gs://compliance-bucket
gsutil iam ch -d allAuthenticatedUsers:objectViewer gs://compliance-bucket
```

**Audit Logging:**
```bash
# Enable audit logs for Cloud Storage
gcloud logging sinks create storage-audit-sink \
    bigquery.googleapis.com/projects/my-project/datasets/audit_logs \
    --log-filter='protoPayload.serviceName="storage.googleapis.com"'
```

**Compliance Monitoring:**
```python
#!/usr/bin/env python3
# compliance-monitor.py

from google.cloud import storage, logging
import json

class ComplianceMonitor:
    def __init__(self, project_id):
        self.storage_client = storage.Client(project=project_id)
        self.logging_client = logging.Client(project=project_id)
    
    def check_public_access(self):
        """Check for buckets with public access."""
        
        public_buckets = []
        
        for bucket in self.storage_client.list_buckets():
            policy = bucket.get_iam_policy()
            
            for binding in policy.bindings:
                if 'allUsers' in binding['members'] or 'allAuthenticatedUsers' in binding['members']:
                    public_buckets.append({
                        "bucket": bucket.name,
                        "role": binding['role'],
                        "members": binding['members']
                    })
        
        return public_buckets
    
    def check_encryption_compliance(self):
        """Check encryption compliance across buckets."""
        
        compliance_report = []
        
        for bucket in self.storage_client.list_buckets():
            bucket_info = {
                "bucket": bucket.name,
                "default_kms_key": bucket.default_kms_key_name,
                "compliant": bucket.default_kms_key_name is not None,
                "unencrypted_objects": []
            }
            
            # Check individual objects
            for blob in bucket.list_blobs(max_results=100):  # Sample check
                if not blob.kms_key_name and not blob.customer_encryption:
                    bucket_info["unencrypted_objects"].append(blob.name)
                    bucket_info["compliant"] = False
            
            compliance_report.append(bucket_info)
        
        return compliance_report
    
    def generate_compliance_report(self):
        """Generate comprehensive compliance report."""
        
        report = {
            "timestamp": datetime.utcnow().isoformat(),
            "public_access_violations": self.check_public_access(),
            "encryption_compliance": self.check_encryption_compliance(),
            "recommendations": []
        }
        
        # Add recommendations
        if report["public_access_violations"]:
            report["recommendations"].append("Remove public access from buckets")
        
        non_compliant_encryption = [b for b in report["encryption_compliance"] if not b["compliant"]]
        if non_compliant_encryption:
            report["recommendations"].append("Enable KMS encryption for all buckets")
        
        return report

# Usage
monitor = ComplianceMonitor("my-project")
report = monitor.generate_compliance_report()
print(json.dumps(report, indent=2))
```

---

## 🔹 Data Transfer Options

### Online Uploads

**gsutil Performance Optimization:**
```bash
# Parallel uploads
gsutil -m cp -r local-directory/ gs://my-bucket/

# Resumable uploads for large files
gsutil -o "GSUtil:resumable_threshold=8388608" cp large-file.zip gs://my-bucket/

# Optimize for throughput
gsutil -o "GSUtil:parallel_thread_count=24" \
       -o "GSUtil:parallel_process_count=4" \
       -m cp -r data/ gs://my-bucket/
```

**Streaming Uploads:**
```python
#!/usr/bin/env python3
# streaming-upload.py

from google.cloud import storage
import io

def stream_upload(bucket_name, destination_name, data_stream):
    """Upload data from stream."""
    
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(destination_name)
    
    # Upload from stream
    blob.upload_from_file(data_stream)
    
    print(f"Streamed data to {destination_name}")

def chunked_upload(bucket_name, destination_name, file_path, chunk_size=8*1024*1024):
    """Upload large file in chunks."""
    
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(destination_name)
    
    # Enable resumable uploads
    blob.chunk_size = chunk_size
    
    with open(file_path, 'rb') as file_obj:
        blob.upload_from_file(file_obj)
    
    print(f"Uploaded {file_path} in chunks")

# Usage
with open('data.json', 'rb') as f:
    stream_upload('my-bucket', 'streamed-data.json', f)

chunked_upload('my-bucket', 'large-file.zip', '/path/to/large-file.zip')
```

### Transfer Service

**Create Transfer Job:**
```bash
# Transfer from AWS S3
gcloud transfer jobs create s3://source-bucket gs://destination-bucket \
    --source-creds-file=aws-credentials.json \
    --description="S3 to GCS migration"

# Transfer from HTTP/HTTPS
gcloud transfer jobs create https://example.com/data/ gs://my-bucket/imported/ \
    --description="HTTP data import"

# Scheduled transfer
gcloud transfer jobs create gs://source-bucket gs://destination-bucket \
    --schedule-start-date=2024-01-01 \
    --schedule-repeats-every=24h \
    --description="Daily backup sync"
```

**Transfer Service Automation:**
```python
#!/usr/bin/env python3
# transfer-automation.py

from google.cloud import storage_transfer
import json

class TransferManager:
    def __init__(self, project_id):
        self.client = storage_transfer.StorageTransferServiceClient()
        self.project_id = project_id
    
    def create_s3_transfer(self, aws_bucket, gcs_bucket, aws_credentials):
        """Create transfer job from AWS S3 to GCS."""
        
        transfer_job = {
            'description': f'Transfer from {aws_bucket} to {gcs_bucket}',
            'project_id': self.project_id,
            'transfer_spec': {
                'aws_s3_data_source': {
                    'bucket_name': aws_bucket,
                    'aws_access_key': aws_credentials
                },
                'gcs_data_sink': {
                    'bucket_name': gcs_bucket
                },
                'transfer_options': {
                    'overwrite_objects_already_existing_in_sink': False,
                    'delete_objects_unique_in_sink': False
                }
            },
            'status': 'ENABLED'
        }
        
        response = self.client.create_transfer_job(
            request={'transfer_job': transfer_job}
        )
        
        print(f"Created transfer job: {response.name}")
        return response
    
    def create_scheduled_backup(self, source_bucket, dest_bucket, schedule):
        """Create scheduled backup transfer."""
        
        transfer_job = {
            'description': f'Scheduled backup: {source_bucket} -> {dest_bucket}',
            'project_id': self.project_id,
            'transfer_spec': {
                'gcs_data_source': {
                    'bucket_name': source_bucket
                },
                'gcs_data_sink': {
                    'bucket_name': dest_bucket
                }
            },
            'schedule': schedule,
            'status': 'ENABLED'
        }
        
        response = self.client.create_transfer_job(
            request={'transfer_job': transfer_job}
        )
        
        return response
    
    def monitor_transfer_operations(self, job_name):
        """Monitor transfer operations."""
        
        operations = self.client.list_transfer_operations(
            request={
                'name': 'transferOperations',
                'filter': json.dumps({'project_id': self.project_id, 'job_names': [job_name]})
            }
        )
        
        for operation in operations:
            print(f"Operation: {operation.name}")
            print(f"Status: {operation.metadata.get('status', 'Unknown')}")
            
            if operation.metadata.get('counters'):
                counters = operation.metadata['counters']
                print(f"Objects copied: {counters.get('objects_copied_to_sink', 0)}")
                print(f"Bytes copied: {counters.get('bytes_copied_to_sink', 0)}")

# Usage
transfer = TransferManager("my-project")

# Daily backup schedule
schedule = {
    'schedule_start_date': {'year': 2024, 'month': 1, 'day': 1},
    'start_time_of_day': {'hours': 2, 'minutes': 0}
}

job = transfer.create_scheduled_backup("production-data", "backup-bucket", schedule)
```

### Transfer Appliance

**Transfer Appliance Workflow:**
```bash
# 1. Order Transfer Appliance
gcloud transfer appliances orders create \
    --display-name="Data Migration Appliance" \
    --type=TA40 \
    --region=us-central1

# 2. Prepare data on appliance (when received)
# - Connect appliance to network
# - Copy data using provided tools
# - Verify data integrity

# 3. Ship back to Google
# - Use provided shipping labels
# - Track shipment status

# 4. Monitor data upload
gcloud transfer operations list \
    --filter="metadata.type=TRANSFER_APPLIANCE"
```

**Large Dataset Migration Strategy:**
```python
#!/usr/bin/env python3
# migration-strategy.py

class MigrationPlanner:
    def __init__(self):
        self.transfer_methods = {
            'online': {'max_size_tb': 10, 'time_days': lambda tb: tb * 2},
            'transfer_service': {'max_size_tb': 100, 'time_days': lambda tb: tb * 0.5},
            'transfer_appliance': {'max_size_tb': 1000, 'time_days': lambda tb: 14}
        }
    
    def recommend_transfer_method(self, data_size_tb, timeline_days, bandwidth_gbps=1):
        """Recommend optimal transfer method."""
        
        recommendations = []
        
        # Calculate online transfer time
        online_time = (data_size_tb * 8000) / (bandwidth_gbps * 86400)  # Days
        
        for method, specs in self.transfer_methods.items():
            if data_size_tb <= specs['max_size_tb']:
                estimated_time = specs['time_days'](data_size_tb)
                
                if method == 'online':
                    estimated_time = online_time
                
                recommendations.append({
                    'method': method,
                    'estimated_time_days': estimated_time,
                    'feasible': estimated_time <= timeline_days,
                    'cost_estimate': self.estimate_cost(method, data_size_tb)
                })
        
        return sorted(recommendations, key=lambda x: x['estimated_time_days'])
    
    def estimate_cost(self, method, data_size_tb):
        """Estimate transfer costs."""
        
        costs = {
            'online': data_size_tb * 0.12,  # Network egress costs
            'transfer_service': data_size_tb * 0.0125,  # Transfer service costs
            'transfer_appliance': 300 + (data_size_tb * 0.02)  # Appliance + shipping
        }
        
        return costs.get(method, 0)

# Usage
planner = MigrationPlanner()
recommendations = planner.recommend_transfer_method(
    data_size_tb=50, 
    timeline_days=30, 
    bandwidth_gbps=10
)

for rec in recommendations:
    print(f"Method: {rec['method']}")
    print(f"Time: {rec['estimated_time_days']:.1f} days")
    print(f"Cost: ${rec['cost_estimate']:.2f}")
    print(f"Feasible: {rec['feasible']}")
    print("---")
```

---

## 🔹 Best Practices & Tools

### Cloud Storage Best Practices

**Performance Optimization:**
```bash
# Request rate optimization
# Avoid sequential naming patterns
# Good: 2024/01/15/log-abc123.txt
# Bad:  2024/01/15/log-000001.txt

# Use parallel uploads
gsutil -m cp -r directory/ gs://bucket/

# Optimize for large files
gsutil -o "GSUtil:parallel_composite_upload_threshold=150M" \
       -o "GSUtil:parallel_composite_upload_component_size=50M" \
       cp large-file.zip gs://bucket/
```

**Cost Optimization:**
```python
#!/usr/bin/env python3
# cost-optimizer.py

from google.cloud import storage
import datetime

class CostOptimizer:
    def __init__(self, project_id):
        self.client = storage.Client(project=project_id)
    
    def analyze_storage_usage(self, bucket_name):
        """Analyze storage usage patterns."""
        
        bucket = self.client.bucket(bucket_name)
        analysis = {
            'total_objects': 0,
            'total_size_gb': 0,
            'storage_classes': {},
            'age_distribution': {'0-30': 0, '30-90': 0, '90-365': 0, '365+': 0},
            'recommendations': []
        }
        
        now = datetime.datetime.now(datetime.timezone.utc)
        
        for blob in bucket.list_blobs():
            analysis['total_objects'] += 1
            analysis['total_size_gb'] += blob.size / (1024**3)
            
            # Track storage classes
            storage_class = blob.storage_class
            analysis['storage_classes'][storage_class] = analysis['storage_classes'].get(storage_class, 0) + 1
            
            # Analyze age
            age_days = (now - blob.time_created).days
            
            if age_days <= 30:
                analysis['age_distribution']['0-30'] += 1
            elif age_days <= 90:
                analysis['age_distribution']['30-90'] += 1
            elif age_days <= 365:
                analysis['age_distribution']['90-365'] += 1
            else:
                analysis['age_distribution']['365+'] += 1
            
            # Generate recommendations
            if age_days > 30 and storage_class == 'STANDARD':
                analysis['recommendations'].append(f"Move {blob.name} to NEARLINE")
            elif age_days > 90 and storage_class == 'NEARLINE':
                analysis['recommendations'].append(f"Move {blob.name} to COLDLINE")
        
        return analysis
    
    def implement_cost_optimization(self, bucket_name):
        """Implement automated cost optimization."""
        
        # Create lifecycle policy
        lifecycle_rules = [
            {
                "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
                "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
            },
            {
                "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
                "condition": {"age": 90, "matchesStorageClass": ["NEARLINE"]}
            },
            {
                "action": {"type": "SetStorageClass", "storageClass": "ARCHIVE"},
                "condition": {"age": 365, "matchesStorageClass": ["COLDLINE"]}
            }
        ]
        
        bucket = self.client.bucket(bucket_name)
        bucket.lifecycle_rules = lifecycle_rules
        bucket.patch()
        
        print(f"Applied cost optimization lifecycle to {bucket_name}")

# Usage
optimizer = CostOptimizer("my-project")
analysis = optimizer.analyze_storage_usage("my-bucket")
print(f"Total size: {analysis['total_size_gb']:.2f} GB")
print(f"Recommendations: {len(analysis['recommendations'])}")
```

### Using gsutil (Command Line)

**Complete gsutil Command Reference:**

**Configuration Commands:**
```bash
# Configure credentials and settings
gsutil config                    # Interactive configuration
gsutil config -a                 # Configure service account
gsutil config -e                 # Configure existing credentials
gsutil config -o Section:Option Value  # Set specific option
gsutil version                   # Show version info
gsutil version -l                # Long version with dependencies

# Configuration file locations:
# ~/.boto (legacy)
# ~/.config/gcloud/legacy_credentials
# Service account key files
```

**Bucket Operations (Complete Options):**
```bash
# Create bucket with all options
gsutil mb [-p PROJECT_ID] [-c STORAGE_CLASS] [-l LOCATION] \
          [-b on|off] [--retention RETENTION_PERIOD] \
          [--pap enforced|inherited] gs://BUCKET_NAME

# Examples:
gsutil mb -p my-project -c STANDARD -l us-central1 gs://bucket
gsutil mb -c NEARLINE -l us --retention 30d gs://backup-bucket
gsutil mb -b on --pap enforced gs://secure-bucket

# Remove bucket operations
gsutil rb gs://bucket-name              # Remove empty bucket
gsutil rb -r gs://bucket-name           # Remove bucket and contents
gsutil rb -f gs://bucket-name           # Force removal (skip confirmation)

# Bucket configuration
gsutil versioning set on|off gs://bucket
gsutil versioning get gs://bucket
gsutil web set -m index.html -e 404.html gs://bucket
gsutil web get gs://bucket
gsutil cors set cors.json gs://bucket
gsutil cors get gs://bucket
gsutil lifecycle set lifecycle.json gs://bucket
gsutil lifecycle get gs://bucket
gsutil logging set on -b log-bucket -o access-logs/ gs://bucket
gsutil logging get gs://bucket
gsutil notification create -t topic-name -f json gs://bucket
gsutil notification list gs://bucket
gsutil requesterpays set on|off gs://bucket
gsutil requesterpays get gs://bucket
```

**Object Operations (All Options):**
```bash
# Copy/Upload with complete options
gsutil cp [OPTIONS] SRC DST

# Copy options:
-A          # Preserve ACLs
-a          # Preserve all metadata
-e          # Exclude symlinks
-L LOG_FILE # Log file path
-n          # No-clobber (don't overwrite)
-p          # Preserve POSIX attributes
-P          # Preserve source timestamps
-r          # Recursive
-R          # Recursive (same as -r)
-s STORAGE_CLASS  # Set storage class
-v          # Print version info for each file
-z EXT_LIST # Compress files with these extensions
-Z          # Compress all files
-j EXT_LIST # Compress only these extensions
-x PATTERN  # Exclude files matching pattern
-I          # Ignore symlinks
-U          # Skip unsupported object types

# Examples:
gsutil cp -r -Z -n -s NEARLINE local-dir/ gs://bucket/
gsutil cp -z txt,html,css,js website/ gs://bucket/
gsutil cp -x ".*\.tmp$|.*\.log$" data/ gs://bucket/

# Move/Rename operations
gsutil mv [-p] SRC DST              # Move with options
gsutil mv -p gs://bucket/old gs://bucket/new  # Preserve metadata

# Remove operations
gsutil rm [-f] [-r] [-a] URI...     # Remove objects
-f          # Force (no confirmation)
-r          # Recursive
-a          # All versions

# Examples:
gsutil rm -r gs://bucket/directory/
gsutil rm -a gs://bucket/file.txt    # Remove all versions
gsutil rm -f gs://bucket/**          # Force remove all
```

**Listing and Information Commands:**
```bash
# List with all options
gsutil ls [OPTIONS] URI...

# List options:
-a          # Include non-current object versions
-b          # Print bucket info
-d          # List matching prefixes instead of contents
-e          # Include ETag in output
-h          # Print headers
-l          # Long listing format
-L          # Extra long listing (all metadata)
-p PROJECT  # Project to list buckets for
-r          # Recursive listing

# Examples:
gsutil ls -l gs://bucket/           # Long format
gsutil ls -L gs://bucket/file.txt   # All metadata
gsutil ls -a gs://bucket/           # Include all versions
gsutil ls -r gs://bucket/prefix/    # Recursive
gsutil ls -b gs://bucket/           # Bucket info

# Detailed object information
gsutil stat [-a] gs://bucket/object  # Object statistics
gsutil du [-a] [-c] [-e] [-h] [-s] [-X] URI...

# du options:
-a          # Include non-current versions
-c          # Include total
-e          # Exclude folders
-h          # Human readable sizes
-s          # Summary only
-X          # Include metadata size
```

**Synchronization (rsync) Complete Options:**
```bash
gsutil rsync [OPTIONS] SRC DST

# rsync options:
-a          # Preserve ACLs
-c          # Compare checksums
-C          # Continue on error
-d          # Delete extra files in DST
-e          # Exclude symlinks
-n          # Dry run
-p          # Preserve POSIX attributes
-P          # Preserve timestamps
-r          # Recursive
-U          # Skip unsupported files
-v          # Verbose output
-x PATTERN  # Exclude pattern

# Examples:
gsutil rsync -r -d -c local-dir/ gs://bucket/remote-dir/
gsutil rsync -r -d -x ".*\.tmp$" local/ gs://bucket/
gsutil rsync -n -r -d local/ gs://bucket/  # Dry run
```

**Performance and Parallel Operations:**
```bash
# Parallel operations
gsutil -m COMMAND [OPTIONS]         # Multi-threaded/parallel

# Performance tuning options:
gsutil -o "GSUtil:parallel_thread_count=24" -m cp -r data/ gs://bucket/
gsutil -o "GSUtil:parallel_process_count=4" -m cp -r data/ gs://bucket/
gsutil -o "GSUtil:resumable_threshold=8388608" cp large-file gs://bucket/
gsutil -o "GSUtil:sliced_object_download_threshold=50M" cp gs://bucket/large-file .

# Configuration options:
GSUtil:parallel_thread_count=N      # Threads per process
GSUtil:parallel_process_count=N     # Number of processes
GSUtil:resumable_threshold=N        # Resumable upload threshold
GSUtil:sliced_object_download_threshold=N  # Sliced download threshold
GSUtil:sliced_object_download_max_components=N  # Max download slices
GSUtil:check_hashes=if_fast_else_skip  # Hash checking
GSUtil:content_language=LANG        # Default content language
GSUtil:default_project_id=PROJECT   # Default project
```

**Access Control Commands:**
```bash
# IAM operations
gsutil iam get gs://bucket/                    # Get IAM policy
gsutil iam set policy.json gs://bucket/        # Set IAM policy
gsutil iam ch MEMBER:ROLE gs://bucket/         # Change IAM binding
gsutil iam ch -d MEMBER:ROLE gs://bucket/      # Delete IAM binding

# IAM examples:
gsutil iam ch user:john@example.com:objectViewer gs://bucket/
gsutil iam ch group:team@example.com:objectAdmin gs://bucket/
gsutil iam ch serviceAccount:sa@project.iam.gserviceaccount.com:objectCreator gs://bucket/
gsutil iam ch -d allUsers:objectViewer gs://bucket/

# ACL operations (legacy)
gsutil acl get gs://bucket/object              # Get object ACL
gsutil acl set acl.xml gs://bucket/object      # Set object ACL
gsutil acl ch -u user@example.com:READ gs://bucket/object
gsutil acl ch -g group@example.com:WRITE gs://bucket/object
gsutil acl ch -d user@example.com gs://bucket/object

# Default ACL operations
gsutil defacl get gs://bucket/                 # Get default ACL
gsutil defacl set acl.xml gs://bucket/         # Set default ACL
gsutil defacl ch -u user@example.com:READ gs://bucket/
```

**Metadata and Header Operations:**
```bash
# Set metadata with all options
gsutil setmeta [OPTIONS] -h "HEADER:VALUE" gs://bucket/object

# Common headers:
-h "Content-Type:application/json"
-h "Content-Language:en"
-h "Content-Encoding:gzip"
-h "Content-Disposition:attachment; filename=file.txt"
-h "Cache-Control:public, max-age=3600"
-h "x-goog-meta-custom-key:custom-value"

# Examples:
gsutil setmeta \
  -h "Content-Type:application/json" \
  -h "Cache-Control:public, max-age=86400" \
  -h "x-goog-meta-department:engineering" \
  gs://bucket/data.json

# Remove metadata (set to empty)
gsutil setmeta -h "x-goog-meta-old-field:" gs://bucket/object
```

**Advanced gsutil Scripting with All Options:**
```bash
#!/bin/bash
# comprehensive-gsutil-operations.sh

BUCKET="gs://my-production-bucket"
LOCAL_DIR="/data/backups"
LOG_FILE="/var/log/gsutil-sync.log"
CONFIG_FILE="/etc/gsutil.conf"

# Advanced gsutil configuration
setup_gsutil_config() {
    cat > $CONFIG_FILE << EOF
[GSUtil]
parallel_thread_count = 24
parallel_process_count = 4
resumable_threshold = 8388608
sliced_object_download_threshold = 52428800
sliced_object_download_max_components = 4
check_hashes = if_fast_else_skip
default_project_id = my-project
content_language = en
prefer_api = json

[Boto]
https_validate_certificates = True
max_retry_delay = 32
num_retries = 6
EOF
}

# Function to log with timestamp and level
log() {
    local level=$1
    local message=$2
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$level] - $message" | tee -a $LOG_FILE
}

# Enhanced sync with all options
enhanced_sync() {
    log "INFO" "Starting enhanced sync to $BUCKET"
    
    # Pre-sync validation
    if ! gsutil ls $BUCKET/ >/dev/null 2>&1; then
        log "ERROR" "Cannot access bucket $BUCKET"
        return 1
    fi
    
    # Sync with comprehensive options
    gsutil -o "GSUtil:parallel_thread_count=24" \
           -o "GSUtil:parallel_process_count=4" \
           -o "GSUtil:resumable_threshold=8388608" \
           -m rsync -r -d -C -c -P \
           -x ".*\.(tmp|log|cache)$" \
           $LOCAL_DIR $BUCKET/backups/ 2>&1 | tee -a $LOG_FILE
    
    local sync_result=${PIPESTATUS[0]}
    
    if [ $sync_result -eq 0 ]; then
        log "INFO" "Sync completed successfully"
        verify_sync_integrity
    else
        log "ERROR" "Sync failed with exit code $sync_result"
        return 1
    fi
}

# Comprehensive sync verification
verify_sync_integrity() {
    log "INFO" "Verifying sync integrity"
    
    # Count files
    local local_count=$(find $LOCAL_DIR -type f ! -name "*.tmp" ! -name "*.log" | wc -l)
    local remote_count=$(gsutil ls -r $BUCKET/backups/ | grep -v "/$" | wc -l)
    
    # Check sizes
    local local_size=$(du -sb $LOCAL_DIR | cut -f1)
    local remote_size=$(gsutil du -s $BUCKET/backups/ | awk '{print $1}')
    
    log "INFO" "Local: $local_count files, $local_size bytes"
    log "INFO" "Remote: $remote_count files, $remote_size bytes"
    
    if [ $local_count -eq $remote_count ] && [ $local_size -eq $remote_size ]; then
        log "INFO" "Verification passed: All files synced correctly"
    else
        log "WARNING" "Verification failed - Size or count mismatch"
        return 1
    fi
}

# Advanced cleanup with lifecycle simulation
advanced_cleanup() {
    log "INFO" "Starting advanced cleanup"
    
    # Cleanup by age with different policies
    local temp_cutoff=$(date -d "1 day ago" +%Y-%m-%dT%H:%M:%SZ)
    local log_cutoff=$(date -d "7 days ago" +%Y-%m-%dT%H:%M:%SZ)
    local backup_cutoff=$(date -d "30 days ago" +%Y-%m-%dT%H:%M:%SZ)
    
    # Remove temporary files
    gsutil -m rm gs://$BUCKET/temp/** 2>/dev/null || true
    
    # Remove old logs
    gsutil ls -l $BUCKET/logs/ | \
    awk -v cutoff="$log_cutoff" '$2 < cutoff {print $3}' | \
    xargs -r gsutil -m rm
    
    # Archive old backups to cheaper storage
    gsutil ls -l $BUCKET/backups/ | \
    awk -v cutoff="$backup_cutoff" '$2 < cutoff {print $3}' | \
    while read object; do
        if [ ! -z "$object" ]; then
            log "INFO" "Moving to COLDLINE: $object"
            gsutil rewrite -s COLDLINE "$object"
        fi
    done
}

# Performance monitoring with detailed metrics
comprehensive_performance_test() {
    log "INFO" "Running comprehensive performance test"
    
    local test_sizes=("1M" "10M" "100M" "1G")
    local results_file="/tmp/gsutil-performance-$(date +%Y%m%d-%H%M%S).log"
    
    echo "Test Results - $(date)" > $results_file
    echo "Size,Upload_Time,Download_Time,Upload_Speed,Download_Speed" >> $results_file
    
    for size in "${test_sizes[@]}"; do
        local test_file="/tmp/test-${size}.dat"
        local remote_path="$BUCKET/performance-test/test-${size}.dat"
        
        # Create test file
        dd if=/dev/urandom of=$test_file bs=$size count=1 2>/dev/null
        local file_size=$(stat -f%z $test_file 2>/dev/null || stat -c%s $test_file)
        
        # Test upload
        local upload_start=$(date +%s.%N)
        gsutil -o "GSUtil:parallel_composite_upload_threshold=0" \
               cp $test_file $remote_path
        local upload_end=$(date +%s.%N)
        local upload_time=$(echo "$upload_end - $upload_start" | bc)
        local upload_speed=$(echo "scale=2; $file_size / $upload_time / 1024 / 1024" | bc)
        
        # Test download
        rm $test_file
        local download_start=$(date +%s.%N)
        gsutil cp $remote_path $test_file
        local download_end=$(date +%s.%N)
        local download_time=$(echo "$download_end - $download_start" | bc)
        local download_speed=$(echo "scale=2; $file_size / $download_time / 1024 / 1024" | bc)
        
        # Log results
        echo "$size,$upload_time,$download_time,${upload_speed}MB/s,${download_speed}MB/s" >> $results_file
        log "INFO" "$size: Upload ${upload_speed}MB/s, Download ${download_speed}MB/s"
        
        # Cleanup
        rm $test_file
        gsutil rm $remote_path
    done
    
    log "INFO" "Performance test completed. Results in $results_file"
}

# Error handling and retry logic
retry_operation() {
    local max_attempts=$1
    local delay=$2
    shift 2
    local command="$@"
    
    for ((i=1; i<=max_attempts; i++)); do
        log "INFO" "Attempt $i/$max_attempts: $command"
        
        if eval $command; then
            log "INFO" "Command succeeded on attempt $i"
            return 0
        else
            log "WARNING" "Command failed on attempt $i"
            if [ $i -lt $max_attempts ]; then
                log "INFO" "Retrying in $delay seconds..."
                sleep $delay
                delay=$((delay * 2))  # Exponential backoff
            fi
        fi
    done
    
    log "ERROR" "Command failed after $max_attempts attempts"
    return 1
}

# Main execution with error handling
main() {
    setup_gsutil_config
    
    # Execute operations with retry logic
    retry_operation 3 5 "enhanced_sync" || exit 1
    retry_operation 2 3 "advanced_cleanup" || log "WARNING" "Cleanup had issues"
    comprehensive_performance_test
    
    log "INFO" "All operations completed successfully"
}

main "$@"
```

### Real-World Scenarios

**Scenario 1: Static Website Hosting**
```bash
#!/bin/bash
# static-website-setup.sh

BUCKET_NAME="my-website-bucket"
DOMAIN="www.example.com"

# Create bucket
gsutil mb gs://$BUCKET_NAME

# Enable website configuration
gsutil web set -m index.html -e 404.html gs://$BUCKET_NAME

# Upload website files
gsutil -m cp -r website/* gs://$BUCKET_NAME/

# Set public access
gsutil iam ch allUsers:objectViewer gs://$BUCKET_NAME

# Set cache headers
gsutil -m setmeta -h "Cache-Control:public, max-age=3600" gs://$BUCKET_NAME/*.html
gsutil -m setmeta -h "Cache-Control:public, max-age=86400" gs://$BUCKET_NAME/assets/*

# Setup CDN (requires load balancer)
gcloud compute backend-buckets create website-backend --gcs-bucket-name=$BUCKET_NAME
```

**Scenario 2: Data Lake Implementation**
```python
#!/usr/bin/env python3
# data-lake-setup.py

from google.cloud import storage
import json

class DataLakeManager:
    def __init__(self, project_id):
        self.client = storage.Client(project=project_id)
        self.project_id = project_id
    
    def setup_data_lake(self, lake_name):
        """Setup multi-tier data lake."""
        
        # Create buckets for different data tiers
        buckets = {
            f"{lake_name}-raw": "STANDARD",      # Raw ingestion
            f"{lake_name}-processed": "NEARLINE", # Processed data
            f"{lake_name}-analytics": "COLDLINE", # Analytics results
            f"{lake_name}-archive": "ARCHIVE"     # Long-term storage
        }
        
        for bucket_name, storage_class in buckets.items():
            bucket = self.client.bucket(bucket_name)
            bucket.storage_class = storage_class
            bucket = self.client.create_bucket(bucket, location="US")
            
            # Set lifecycle policies
            self.set_data_lifecycle(bucket_name)
            
            print(f"Created {bucket_name} with {storage_class} storage")
    
    def set_data_lifecycle(self, bucket_name):
        """Set appropriate lifecycle for data tier."""
        
        bucket = self.client.bucket(bucket_name)
        
        if "raw" in bucket_name:
            # Raw data: Standard -> Nearline -> Coldline
            lifecycle_rules = [
                {
                    "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
                    "condition": {"age": 7, "matchesStorageClass": ["STANDARD"]}
                },
                {
                    "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
                    "condition": {"age": 30, "matchesStorageClass": ["NEARLINE"]}
                }
            ]
        elif "processed" in bucket_name:
            # Processed data: Keep in Nearline, archive after 1 year
            lifecycle_rules = [
                {
                    "action": {"type": "SetStorageClass", "storageClass": "ARCHIVE"},
                    "condition": {"age": 365, "matchesStorageClass": ["NEARLINE"]}
                }
            ]
        else:
            # Analytics and archive: No automatic transitions
            lifecycle_rules = []
        
        bucket.lifecycle_rules = lifecycle_rules
        bucket.patch()
    
    def ingest_data(self, source_path, data_type, metadata=None):
        """Ingest data into appropriate tier."""
        
        # Determine target bucket based on data type
        if data_type == "raw":
            bucket_name = f"{self.lake_name}-raw"
            prefix = f"raw/{datetime.now().strftime('%Y/%m/%d')}/"
        elif data_type == "processed":
            bucket_name = f"{self.lake_name}-processed"
            prefix = f"processed/{datetime.now().strftime('%Y/%m')}/"
        
        bucket = self.client.bucket(bucket_name)
        destination_name = f"{prefix}{os.path.basename(source_path)}"
        blob = bucket.blob(destination_name)
        
        # Set metadata
        if metadata:
            blob.metadata = metadata
        
        # Upload file
        blob.upload_from_filename(source_path)
        
        print(f"Ingested {source_path} to {bucket_name}/{destination_name}")

# Usage
lake = DataLakeManager("my-project")
lake.setup_data_lake("analytics-lake")
```

**Scenario 3: Backup and Disaster Recovery**
```bash
#!/bin/bash
# backup-dr-system.sh

PROJECT_ID="my-project"
PRIMARY_REGION="us-central1"
DR_REGION="us-east1"

# Setup primary backup bucket
gsutil mb -p $PROJECT_ID -c STANDARD -l $PRIMARY_REGION gs://backup-primary-$PROJECT_ID

# Setup DR backup bucket
gsutil mb -p $PROJECT_ID -c NEARLINE -l $DR_REGION gs://backup-dr-$PROJECT_ID

# Enable versioning on both buckets
gsutil versioning set on gs://backup-primary-$PROJECT_ID
gsutil versioning set on gs://backup-dr-$PROJECT_ID

# Set retention policies
gsutil retention set 2555d gs://backup-primary-$PROJECT_ID  # 7 years
gsutil retention set 2555d gs://backup-dr-$PROJECT_ID

# Create cross-region replication
gcloud transfer jobs create gs://backup-primary-$PROJECT_ID gs://backup-dr-$PROJECT_ID \
    --schedule-repeats-every=24h \
    --description="Daily DR replication"

# Backup function
backup_database() {
    local db_name=$1
    local backup_file="db-backup-$(date +%Y%m%d-%H%M%S).sql"
    
    # Create database backup
    pg_dump $db_name > /tmp/$backup_file
    
    # Compress backup
    gzip /tmp/$backup_file
    
    # Upload to primary bucket
    gsutil cp /tmp/${backup_file}.gz gs://backup-primary-$PROJECT_ID/database/
    
    # Cleanup local file
    rm /tmp/${backup_file}.gz
    
    echo "Database backup completed: ${backup_file}.gz"
}

# Restore function
restore_database() {
    local backup_file=$1
    local db_name=$2
    
    # Download backup
    gsutil cp gs://backup-primary-$PROJECT_ID/database/$backup_file /tmp/
    
    # Decompress and restore
    gunzip /tmp/$backup_file
    psql $db_name < /tmp/${backup_file%.gz}
    
    # Cleanup
    rm /tmp/${backup_file%.gz}
    
    echo "Database restored from: $backup_file"
}

# Test DR failover
test_dr_failover() {
    echo "Testing DR failover..."
    
    # Verify DR bucket accessibility
    gsutil ls gs://backup-dr-$PROJECT_ID/ > /dev/null
    
    if [ $? -eq 0 ]; then
        echo "DR bucket accessible"
        
        # Test restore from DR bucket
        LATEST_BACKUP=$(gsutil ls gs://backup-dr-$PROJECT_ID/database/ | tail -1)
        echo "Latest DR backup: $LATEST_BACKUP"
    else
        echo "ERROR: DR bucket not accessible"
        exit 1
    fi
}

# Main backup routine
main() {
    backup_database "production_db"
    test_dr_failover
}

main "$@"
```

This comprehensive Google Cloud Storage guide provides you with the knowledge and practical examples needed to master Cloud Storage from a DevOps perspective. The guide covers everything from basic concepts to advanced implementation patterns, security, compliance, and real-world scenarios.
# 🔐 Data Security & Encryption in GCP
*DevOps-Focused Complete Reference*

## Table of Contents
1. [Data States](#data-states)
2. [Encryption Basics](#encryption-basics)
3. [Symmetric Key Encryption](#symmetric-key-encryption)
4. [Asymmetric Key Encryption](#asymmetric-key-encryption)
5. [Google Cloud Key Management Service (KMS)](#google-cloud-key-management-service-kms)
6. [Encryption Implementation](#encryption-implementation)
7. [Security Best Practices](#security-best-practices)
8. [Compliance & Auditing](#compliance--auditing)
9. [DevOps Integration](#devops-integration)
10. [Monitoring & Incident Response](#monitoring--incident-response)

---

## 🔹 Data States

### Data at Rest

**Definition:** Data stored on physical devices, databases, or backup systems when not actively being processed.

**Examples:**
- Files stored on Cloud Storage buckets
- Database records in Cloud SQL/Firestore
- VM disk snapshots and images
- Backup archives and cold storage
- Log files in Cloud Logging

**GCP Services with Data at Rest:**
```bash
# Cloud Storage - automatically encrypted
gsutil cp sensitive-data.txt gs://my-bucket/

# Cloud SQL - encryption enabled by default
gcloud sql instances create my-instance --database-version=POSTGRES_13

# Compute Engine - persistent disks encrypted
gcloud compute disks create my-disk --size=100GB --type=pd-ssd
```

**DevOps Considerations:**
- **Backup encryption** - Ensure backups are encrypted
- **Snapshot security** - Encrypt VM and disk snapshots
- **Archive policies** - Long-term storage encryption requirements
- **Data lifecycle** - Encryption throughout data retention periods

### Data in Motion (Transit)

**Definition:** Data actively moving between locations, services, or networks.

**Types of Data in Transit:**

**1. Into/Out of Cloud (Internet Traffic):**
- Client applications to GCP services
- On-premises to cloud data transfers
- Third-party API communications
- User uploads and downloads

**2. Within Cloud (Internal Traffic):**
- Service-to-service communication
- Inter-region data replication
- Load balancer to backend traffic
- Database connections

**Examples:**
```bash
# HTTPS traffic (encrypted in transit)
curl -X POST https://api.myapp.com/data \
  -H "Content-Type: application/json" \
  -d '{"sensitive": "data"}'

# VPC internal traffic (can be encrypted)
gcloud compute instances create web-server \
  --subnet=private-subnet \
  --no-address  # Internal traffic only
```

**DevOps Implementation:**
```yaml
# Load balancer with SSL termination
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: ssl-certificate
spec:
  domains:
    - api.mycompany.com
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    networking.gke.io/managed-certificates: ssl-certificate
    kubernetes.io/ingress.class: "gce"
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
```

### Data in Use

**Definition:** Data actively being processed in memory (RAM) or CPU registers.

**Examples:**
- Application variables and session data
- Database query results in memory
- Machine learning model computations
- Temporary processing buffers

**Security Challenges:**
- Memory dumps can expose sensitive data
- Process isolation is critical
- Confidential computing for sensitive workloads

**GCP Confidential Computing:**
```bash
# Create confidential VM
gcloud compute instances create confidential-vm \
  --zone=us-central1-a \
  --machine-type=n2d-standard-2 \
  --confidential-compute \
  --maintenance-policy=TERMINATE
```

**DevOps Considerations:**
- **Memory management** - Clear sensitive data from memory
- **Process isolation** - Container and VM security
- **Confidential workloads** - Use Confidential GKE for sensitive processing
- **Application security** - Secure coding practices

---

## 🔹 Encryption Basics

### Why Encrypt?

**Defense in Depth Principle:**
Encryption provides protection even if other security controls fail.

**Risk Scenarios:**
```bash
# Unencrypted disk theft scenario
# Attacker gains physical access to storage device
# → All data readable without encryption

# Network interception scenario  
# Attacker intercepts network traffic
# → Sensitive data exposed without TLS/SSL

# Database breach scenario
# Attacker gains database access
# → Encrypted columns remain protected
```

**Encryption Benefits:**
- **Confidentiality** - Prevents unauthorized data access
- **Compliance** - Meets regulatory requirements (GDPR, HIPAA, PCI-DSS)
- **Risk mitigation** - Reduces impact of security breaches
- **Trust** - Builds customer confidence

### Encryption at All Stages

**Comprehensive Encryption Strategy:**
```yaml
# Data lifecycle encryption
Data Creation:
  - Encrypt at source
  - Use secure key generation
  
Data Storage:
  - Encrypt databases
  - Encrypt file systems
  - Encrypt backups
  
Data Transit:
  - TLS/SSL for web traffic
  - VPN for network connections
  - Encrypted API communications
  
Data Processing:
  - Confidential computing
  - Secure enclaves
  - Memory encryption
  
Data Archival:
  - Long-term key management
  - Encrypted cold storage
  - Secure key escrow
```

**DevOps Implementation Matrix:**
| Data State | GCP Service | Encryption Method | Key Management |
|------------|-------------|-------------------|----------------|
| **At Rest** | Cloud Storage | AES-256 | Google/CMEK/CSEK |
| **At Rest** | Cloud SQL | AES-256 | Google/CMEK |
| **In Transit** | Load Balancer | TLS 1.2/1.3 | Managed Certificates |
| **In Transit** | VPC | IPSec (optional) | Cloud VPN |
| **In Use** | Confidential GKE | Memory encryption | Hardware TEE |

---

## 🔹 Symmetric Key Encryption

### Concept Overview

**Symmetric Encryption:** Same key used for both encryption and decryption operations.

**Key Characteristics:**
- **Fast performance** - Efficient for large data volumes
- **Shared secret** - Both parties need the same key
- **Key distribution challenge** - Securely sharing keys

### Algorithm Selection

**Recommended Algorithms:**
```python
# AES (Advanced Encryption Standard) - Industry standard
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
import os

# AES-256 example
def encrypt_aes_256(plaintext, key):
    """Encrypt data using AES-256-GCM."""
    # Generate random IV
    iv = os.urandom(12)  # 96-bit IV for GCM
    
    # Create cipher
    cipher = Cipher(
        algorithms.AES(key),
        modes.GCM(iv)
    )
    
    encryptor = cipher.encryptor()
    ciphertext = encryptor.update(plaintext.encode()) + encryptor.finalize()
    
    return {
        'ciphertext': ciphertext,
        'iv': iv,
        'tag': encryptor.tag
    }

def decrypt_aes_256(encrypted_data, key):
    """Decrypt AES-256-GCM encrypted data."""
    cipher = Cipher(
        algorithms.AES(key),
        modes.GCM(encrypted_data['iv'], encrypted_data['tag'])
    )
    
    decryptor = cipher.decryptor()
    plaintext = decryptor.update(encrypted_data['ciphertext']) + decryptor.finalize()
    
    return plaintext.decode()

# Generate secure key
key = os.urandom(32)  # 256-bit key
```

### Key Management Challenges

**Secure Key Storage:**
```python
# Bad practice - hardcoded keys
SECRET_KEY = "hardcoded-key-123"  # Never do this!

# Good practice - use Cloud KMS
from google.cloud import kms

def get_encryption_key():
    """Retrieve encryption key from Cloud KMS."""
    client = kms.KeyManagementServiceClient()
    
    # Key resource name
    key_name = client.crypto_key_path(
        project='my-project',
        location='global',
        key_ring='my-keyring',
        crypto_key='my-key'
    )
    
    # Generate data encryption key
    response = client.generate_random_bytes(request={'location': 'global', 'length_bytes': 32})
    return response.data

def encrypt_with_kms(plaintext, key_name):
    """Encrypt data using Cloud KMS."""
    client = kms.KeyManagementServiceClient()
    
    response = client.encrypt(
        request={
            'name': key_name,
            'plaintext': plaintext.encode()
        }
    )
    
    return response.ciphertext
```

**Key Rotation Strategy:**
```bash
# Automated key rotation with Cloud KMS
gcloud kms keys create my-key \
    --keyring=my-keyring \
    --location=global \
    --purpose=encryption \
    --rotation-period=90d \
    --next-rotation-time=2024-04-01T00:00:00Z
```

### DevOps Key Management

**Infrastructure as Code:**
```hcl
# Terraform KMS configuration
resource "google_kms_key_ring" "keyring" {
  name     = "my-keyring"
  location = "global"
}

resource "google_kms_crypto_key" "key" {
  name     = "my-encryption-key"
  key_ring = google_kms_key_ring.keyring.id
  
  rotation_period = "7776000s"  # 90 days
  
  lifecycle {
    prevent_destroy = true
  }
}

resource "google_kms_crypto_key_iam_binding" "crypto_key" {
  crypto_key_id = google_kms_crypto_key.key.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  
  members = [
    "serviceAccount:${google_service_account.app.email}",
  ]
}
```

---

## 🔹 Asymmetric Key Encryption

### Public Key Cryptography

**Concept:** Uses a pair of mathematically related keys - public and private.

**Key Properties:**
- **Public Key** - Can be shared openly, used for encryption
- **Private Key** - Must be kept secret, used for decryption
- **One-way function** - Easy to compute forward, hard to reverse

### Use Cases

**1. Secure Key Exchange:**
```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes, serialization

# Generate RSA key pair
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048
)
public_key = private_key.public_key()

def secure_key_exchange(symmetric_key, recipient_public_key):
    """Encrypt symmetric key with recipient's public key."""
    encrypted_key = recipient_public_key.encrypt(
        symmetric_key,
        padding.OAEP(
            mgf=padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )
    return encrypted_key

def decrypt_symmetric_key(encrypted_key, private_key):
    """Decrypt symmetric key with private key."""
    symmetric_key = private_key.decrypt(
        encrypted_key,
        padding.OAEP(
            mgf=padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )
    return symmetric_key
```

**2. Digital Signatures:**
```python
def sign_data(data, private_key):
    """Create digital signature."""
    signature = private_key.sign(
        data.encode(),
        padding.PSS(
            mgf=padding.MGF1(hashes.SHA256()),
            salt_length=padding.PSS.MAX_LENGTH
        ),
        hashes.SHA256()
    )
    return signature

def verify_signature(data, signature, public_key):
    """Verify digital signature."""
    try:
        public_key.verify(
            signature,
            data.encode(),
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA256()),
                salt_length=padding.PSS.MAX_LENGTH
            ),
            hashes.SHA256()
        )
        return True
    except Exception:
        return False
```

### TLS/SSL Implementation

**Certificate Management:**
```bash
# Create SSL certificate with Cloud Certificate Manager
gcloud certificate-manager certificates create my-cert \
    --domains="api.mycompany.com,www.mycompany.com" \
    --global

# Configure load balancer with SSL
gcloud compute target-https-proxies create my-https-proxy \
    --ssl-certificates=my-cert \
    --url-map=my-url-map
```

**Application-Level TLS:**
```python
import ssl
import socket

def create_secure_connection(hostname, port):
    """Create TLS-encrypted connection."""
    # Create SSL context
    context = ssl.create_default_context()
    context.check_hostname = True
    context.verify_mode = ssl.CERT_REQUIRED
    
    # Create secure socket
    with socket.create_connection((hostname, port)) as sock:
        with context.wrap_socket(sock, server_hostname=hostname) as ssock:
            # Verify certificate
            cert = ssock.getpeercert()
            print(f"Connected to {cert['subject']}")
            
            # Send encrypted data
            ssock.send(b"GET / HTTP/1.1\r\nHost: " + hostname.encode() + b"\r\n\r\n")
            response = ssock.recv(4096)
            return response.decode()
```

### DevOps Certificate Management

**Automated Certificate Provisioning:**
```yaml
# Kubernetes cert-manager configuration
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-tls
  namespace: production
spec:
  secretName: api-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - api.mycompany.com
  - www.mycompany.com
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - api.mycompany.com
    secretName: api-tls-secret
  rules:
  - host: api.mycompany.com
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

## 🔹 Google Cloud Key Management Service (KMS)

### KMS Overview

**Cloud KMS** is a centralized key management service that allows you to create, manage, and control cryptographic keys across GCP services.

**Key Features:**
- **Centralized key management** - Single point of control
- **Hardware Security Modules (HSM)** - FIPS 140-2 Level 3 validated
- **Automatic key rotation** - Configurable rotation schedules
- **Fine-grained access control** - IAM integration
- **Audit logging** - Complete key usage tracking

### Key Types

**1. Google-Managed Keys (Default):**
```bash
# Automatic encryption with Google-managed keys
gsutil cp sensitive-file.txt gs://my-bucket/
# File is automatically encrypted with Google-managed keys
```

**2. Customer-Managed Encryption Keys (CMEK):**
```bash
# Create key ring and key
gcloud kms keyrings create my-keyring --location=global

gcloud kms keys create my-key \
    --keyring=my-keyring \
    --location=global \
    --purpose=encryption

# Use CMEK with Cloud Storage
gsutil cp sensitive-file.txt gs://my-bucket/ \
    -o "GSUtil:encryption_key=projects/my-project/locations/global/keyRings/my-keyring/cryptoKeys/my-key"
```

**3. Customer-Supplied Encryption Keys (CSEK):**
```bash
# Generate your own key
openssl rand -base64 32 > my-key.txt

# Use CSEK with Compute Engine
gcloud compute disks create my-disk \
    --size=100GB \
    --csek-key-file=my-key.txt
```

### KMS API Integration

**Python SDK Example:**
```python
from google.cloud import kms
import base64

class KMSManager:
    def __init__(self, project_id, location, key_ring, key_name):
        self.client = kms.KeyManagementServiceClient()
        self.key_name = self.client.crypto_key_path(
            project_id, location, key_ring, key_name
        )
    
    def encrypt(self, plaintext):
        """Encrypt data using Cloud KMS."""
        response = self.client.encrypt(
            request={
                'name': self.key_name,
                'plaintext': plaintext.encode()
            }
        )
        return base64.b64encode(response.ciphertext).decode()
    
    def decrypt(self, ciphertext):
        """Decrypt data using Cloud KMS."""
        ciphertext_bytes = base64.b64decode(ciphertext.encode())
        response = self.client.decrypt(
            request={
                'name': self.key_name,
                'ciphertext': ciphertext_bytes
            }
        )
        return response.plaintext.decode()
    
    def generate_data_key(self, key_spec='AES_256'):
        """Generate a data encryption key."""
        response = self.client.generate_random_bytes(
            request={
                'location': self.client.location_path(
                    self.key_name.split('/')[1],
                    self.key_name.split('/')[3]
                ),
                'length_bytes': 32 if key_spec == 'AES_256' else 16
            }
        )
        return response.data

# Usage example
kms = KMSManager('my-project', 'global', 'my-keyring', 'my-key')

# Encrypt sensitive data
encrypted_data = kms.encrypt("sensitive information")
print(f"Encrypted: {encrypted_data}")

# Decrypt data
decrypted_data = kms.decrypt(encrypted_data)
print(f"Decrypted: {decrypted_data}")
```

### Envelope Encryption Pattern

**Implementation:**
```python
import os
from cryptography.fernet import Fernet

class EnvelopeEncryption:
    def __init__(self, kms_manager):
        self.kms = kms_manager
    
    def encrypt_large_data(self, data):
        """Encrypt large data using envelope encryption."""
        # Generate data encryption key (DEK)
        dek = Fernet.generate_key()
        
        # Encrypt data with DEK
        f = Fernet(dek)
        encrypted_data = f.encrypt(data.encode())
        
        # Encrypt DEK with KMS (Key Encryption Key)
        encrypted_dek = self.kms.encrypt(dek.decode())
        
        return {
            'encrypted_data': encrypted_data,
            'encrypted_dek': encrypted_dek
        }
    
    def decrypt_large_data(self, envelope):
        """Decrypt data using envelope encryption."""
        # Decrypt DEK with KMS
        dek = self.kms.decrypt(envelope['encrypted_dek']).encode()
        
        # Decrypt data with DEK
        f = Fernet(dek)
        decrypted_data = f.decrypt(envelope['encrypted_data'])
        
        return decrypted_data.decode()

# Usage
envelope = EnvelopeEncryption(kms)
large_data = "This is a large amount of sensitive data..." * 1000

# Encrypt
encrypted_envelope = envelope.encrypt_large_data(large_data)

# Decrypt
decrypted_data = envelope.decrypt_large_data(encrypted_envelope)
```

### Service Integration

**Cloud SQL with CMEK:**
```bash
# Create Cloud SQL instance with CMEK
gcloud sql instances create my-instance \
    --database-version=POSTGRES_13 \
    --tier=db-f1-micro \
    --region=us-central1 \
    --disk-encryption-key=projects/my-project/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key
```

**Cloud Storage with CMEK:**
```bash
# Set default CMEK for bucket
gsutil kms encryption \
    -k projects/my-project/locations/global/keyRings/my-keyring/cryptoKeys/my-key \
    gs://my-bucket
```

**Compute Engine with CMEK:**
```bash
# Create encrypted disk
gcloud compute disks create my-encrypted-disk \
    --size=100GB \
    --kms-key=projects/my-project/locations/global/keyRings/my-keyring/cryptoKeys/my-key
```

### DevOps KMS Management

**Terraform KMS Setup:**
```hcl
# Complete KMS setup with Terraform
resource "google_kms_key_ring" "keyring" {
  name     = "production-keyring"
  location = "global"
}

resource "google_kms_crypto_key" "database_key" {
  name     = "database-encryption-key"
  key_ring = google_kms_key_ring.keyring.id
  purpose  = "ENCRYPT_DECRYPT"
  
  rotation_period = "7776000s"  # 90 days
  
  version_template {
    algorithm = "GOOGLE_SYMMETRIC_ENCRYPTION"
  }
  
  lifecycle {
    prevent_destroy = true
  }
}

resource "google_kms_crypto_key" "storage_key" {
  name     = "storage-encryption-key"
  key_ring = google_kms_key_ring.keyring.id
  purpose  = "ENCRYPT_DECRYPT"
  
  rotation_period = "2592000s"  # 30 days
}

# IAM bindings for service accounts
resource "google_kms_crypto_key_iam_binding" "database_key_binding" {
  crypto_key_id = google_kms_crypto_key.database_key.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  
  members = [
    "serviceAccount:${google_service_account.database_sa.email}",
  ]
}

resource "google_kms_crypto_key_iam_binding" "storage_key_binding" {
  crypto_key_id = google_kms_crypto_key.storage_key.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  
  members = [
    "serviceAccount:${google_service_account.app_sa.email}",
  ]
}
```

**Key Rotation Automation:**
```python
from google.cloud import kms
import datetime

def rotate_key_if_needed(project_id, location, key_ring, key_name, max_age_days=90):
    """Rotate KMS key if it's older than max_age_days."""
    client = kms.KeyManagementServiceClient()
    
    key_path = client.crypto_key_path(project_id, location, key_ring, key_name)
    key = client.get_crypto_key(request={'name': key_path})
    
    # Get primary version
    primary_version = key.primary
    create_time = primary_version.create_time
    
    # Calculate age
    age = datetime.datetime.now(datetime.timezone.utc) - create_time
    
    if age.days > max_age_days:
        print(f"Key {key_name} is {age.days} days old, rotating...")
        
        # Create new version (automatic rotation)
        new_version = client.create_crypto_key_version(
            request={
                'parent': key_path,
                'crypto_key_version': {}
            }
        )
        
        print(f"Created new key version: {new_version.name}")
        return new_version
    else:
        print(f"Key {key_name} is {age.days} days old, no rotation needed")
        return None

# Automated rotation script
if __name__ == "__main__":
    keys_to_check = [
        ('my-project', 'global', 'prod-keyring', 'database-key'),
        ('my-project', 'global', 'prod-keyring', 'storage-key'),
    ]
    
    for project, location, keyring, key in keys_to_check:
        rotate_key_if_needed(project, location, keyring, key)
```

This comprehensive guide provides you with the knowledge and practical examples needed to implement robust data security and encryption in GCP from a DevOps perspective. The guide covers fundamental concepts through advanced implementation patterns, all tailored for your senior DevOps role.
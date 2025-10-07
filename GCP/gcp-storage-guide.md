# 💾 GCP Storage Complete Guide
*DevOps-Focused Reference*

## Table of Contents
1. [Introduction to Storage in GCP](#introduction-to-storage-in-gcp)
2. [Block Storage](#block-storage)
3. [Snapshots & Images](#snapshots--images)
4. [Persistent Disk Operations](#persistent-disk-operations)
5. [File Storage](#file-storage)
6. [DevOps Best Practices](#devops-best-practices)
7. [Monitoring & Troubleshooting](#monitoring--troubleshooting)

---

## 🔹 Introduction to Storage in GCP

### Block vs File Storage in GCP

**Block Storage:**
- **Raw storage blocks** attached to compute instances
- **Direct access** to storage blocks by OS
- **High performance** for databases and file systems
- **Examples:** Persistent Disks, Local SSDs

**File Storage:**
- **Network-attached storage** with file system interface
- **Shared access** across multiple instances
- **POSIX-compliant** file system
- **Examples:** Filestore (NFS)

```bash
# Block Storage Example - Persistent Disk
gcloud compute disks create my-block-disk \
    --size=100GB \
    --type=pd-ssd \
    --zone=us-central1-a

# File Storage Example - Filestore
gcloud filestore instances create my-nfs-server \
    --zone=us-central1-a \
    --tier=STANDARD \
    --file-share=name="share1",capacity=1TB \
    --network=name="default"
```

### Global, Regional, and Zonal Resources

**Resource Scope Comparison:**

| Scope | Availability | Use Case | Examples |
|-------|-------------|----------|----------|
| **Zonal** | Single zone | High performance, low latency | Local SSDs, Zonal Persistent Disks |
| **Regional** | Multiple zones in region | High availability, disaster recovery | Regional Persistent Disks |
| **Global** | Multiple regions | Global distribution | Cloud Storage, Images |

**DevOps Decision Matrix:**
```yaml
# Choose Zonal for:
- High IOPS requirements
- Low latency applications
- Cost-sensitive workloads

# Choose Regional for:
- High availability requirements
- Cross-zone failover
- Production databases

# Choose Global for:
- Multi-region applications
- Content distribution
- Backup and archival
```

---

## 🔹 Block Storage

### Local SSDs in GCP

**Characteristics:**
- **Physically attached** to the host machine
- **Highest performance** - up to 2.4M IOPS
- **Ephemeral storage** - data lost on instance stop/restart
- **Fixed sizes** - 375GB per device, up to 24 devices (9TB total)

**Use Cases:**
- High-performance databases (MongoDB, Cassandra)
- Caching layers (Redis, Memcached)
- Temporary processing workloads
- Scratch space for data processing

**Implementation:**
```bash
# Create VM with Local SSDs
gcloud compute instances create ssd-instance \
    --zone=us-central1-a \
    --machine-type=n1-standard-4 \
    --local-ssd=interface=SCSI \
    --local-ssd=interface=SCSI \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud

# Format and mount Local SSDs
sudo mkfs.ext4 -F /dev/sdb
sudo mkdir -p /mnt/disks/ssd0
sudo mount /dev/sdb /mnt/disks/ssd0
sudo chmod a+w /mnt/disks/ssd0
```

**RAID Configuration for Local SSDs:**
```bash
#!/bin/bash
# Create RAID 0 for maximum performance
sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
sudo mkfs.ext4 -F /dev/md0
sudo mkdir -p /mnt/disks/raid0
sudo mount /dev/md0 /mnt/disks/raid0

# Add to fstab for persistence
echo '/dev/md0 /mnt/disks/raid0 ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

### Persistent Disks

**Overview:**
- **Network-attached storage** independent of VM lifecycle
- **Durable and reliable** with automatic replication
- **Resizable** while attached to running instances
- **Snapshotable** for backup and cloning

### Persistent Disk Types

**1. Standard Persistent Disk (pd-standard):**
```bash
# Create standard persistent disk
gcloud compute disks create standard-disk \
    --size=500GB \
    --type=pd-standard \
    --zone=us-central1-a

# Performance characteristics
# IOPS: 0.75 per GB (max 7,500)
# Throughput: 120 MB/s per TB (max 1,200 MB/s)
# Cost: Lowest cost option
```

**2. SSD Persistent Disk (pd-ssd):**
```bash
# Create SSD persistent disk
gcloud compute disks create ssd-disk \
    --size=100GB \
    --type=pd-ssd \
    --zone=us-central1-a

# Performance characteristics
# IOPS: 30 per GB (max 100,000)
# Throughput: 480 MB/s per TB (max 1,200 MB/s)
# Cost: Higher cost, better performance
```

**3. Balanced Persistent Disk (pd-balanced):**
```bash
# Create balanced persistent disk
gcloud compute disks create balanced-disk \
    --size=200GB \
    --type=pd-balanced \
    --zone=us-central1-a

# Performance characteristics
# IOPS: 6 per GB (max 80,000)
# Throughput: 240 MB/s per TB (max 1,200 MB/s)
# Cost: Balance between cost and performance
```

**4. Extreme Persistent Disk (pd-extreme):**
```bash
# Create extreme persistent disk
gcloud compute disks create extreme-disk \
    --size=64GB \
    --type=pd-extreme \
    --provisioned-iops=10000 \
    --zone=us-central1-a

# Performance characteristics
# IOPS: Configurable up to 100,000
# Throughput: Up to 4,800 MB/s
# Cost: Highest cost, maximum performance
```

### Performance Comparison

| Disk Type | Max IOPS | Max Throughput | Use Case |
|-----------|----------|----------------|----------|
| **pd-standard** | 7,500 | 1,200 MB/s | Boot disks, file servers |
| **pd-balanced** | 80,000 | 1,200 MB/s | General purpose workloads |
| **pd-ssd** | 100,000 | 1,200 MB/s | High-performance databases |
| **pd-extreme** | 100,000 | 4,800 MB/s | Mission-critical applications |
| **Local SSD** | 2,400,000 | 6,400 MB/s | Ultra-high performance |

### Regional Persistent Disks

**High Availability Setup:**
```bash
# Create regional persistent disk
gcloud compute disks create regional-disk \
    --size=500GB \
    --type=pd-ssd \
    --region=us-central1

# Attach to instance in any zone within region
gcloud compute instances attach-disk my-instance \
    --disk=regional-disk \
    --zone=us-central1-a
```

**DevOps Benefits:**
- **Cross-zone availability** - Survives zone failures
- **Automatic failover** - No manual intervention needed
- **Consistent performance** - Same IOPS across zones
- **Simplified DR** - Built-in disaster recovery

---

## 🔹 Snapshots & Images

### Taking Snapshots for Persistent Disks

**Basic Snapshot Creation:**
```bash
# Create snapshot of persistent disk
gcloud compute disks snapshot my-disk \
    --snapshot-name=my-disk-snapshot-$(date +%Y%m%d-%H%M%S) \
    --zone=us-central1-a \
    --description="Backup before maintenance"

# Create snapshot with labels
gcloud compute disks snapshot my-disk \
    --snapshot-name=production-backup \
    --zone=us-central1-a \
    --labels=environment=production,backup-type=scheduled
```

**Automated Snapshot Scheduling:**
```bash
# Create snapshot schedule
gcloud compute resource-policies create snapshot-schedule daily-backup \
    --region=us-central1 \
    --max-retention-days=30 \
    --on-source-disk-delete=keep-auto-snapshots \
    --daily-schedule \
    --start-time=02:00 \
    --storage-location=us

# Attach schedule to disk
gcloud compute disks add-resource-policies my-disk \
    --resource-policies=daily-backup \
    --zone=us-central1-a
```

**Snapshot Best Practices:**
```python
#!/usr/bin/env python3
# Automated snapshot management script

from google.cloud import compute_v1
import datetime

def create_snapshot_with_retention(project_id, zone, disk_name, retention_days=7):
    """Create snapshot with automatic cleanup."""
    
    client = compute_v1.DisksClient()
    snapshots_client = compute_v1.SnapshotsClient()
    
    # Create snapshot
    timestamp = datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
    snapshot_name = f"{disk_name}-{timestamp}"
    
    operation = client.create_snapshot(
        project=project_id,
        zone=zone,
        disk=disk_name,
        snapshot_resource=compute_v1.Snapshot(
            name=snapshot_name,
            description=f"Automated backup of {disk_name}",
            labels={
                "created-by": "automation",
                "source-disk": disk_name,
                "retention-days": str(retention_days)
            }
        )
    )
    
    print(f"Creating snapshot: {snapshot_name}")
    
    # Cleanup old snapshots
    cleanup_old_snapshots(project_id, disk_name, retention_days)

def cleanup_old_snapshots(project_id, disk_name, retention_days):
    """Remove snapshots older than retention period."""
    
    snapshots_client = compute_v1.SnapshotsClient()
    cutoff_date = datetime.datetime.now() - datetime.timedelta(days=retention_days)
    
    for snapshot in snapshots_client.list(project=project_id):
        if (snapshot.labels.get("source-disk") == disk_name and 
            snapshot.creation_timestamp < cutoff_date.isoformat()):
            
            print(f"Deleting old snapshot: {snapshot.name}")
            snapshots_client.delete(project=project_id, snapshot=snapshot.name)
```

### Machine Images in GCP

**Complete VM Backup:**
```bash
# Create machine image (includes all disks and metadata)
gcloud compute machine-images create my-vm-image \
    --source-instance=my-instance \
    --source-instance-zone=us-central1-a \
    --description="Complete VM backup with all disks"

# Create machine image with specific disks
gcloud compute machine-images create selective-image \
    --source-instance=my-instance \
    --source-instance-zone=us-central1-a \
    --storage-location=us \
    --guest-flush
```

**Restore from Machine Image:**
```bash
# Create new instance from machine image
gcloud compute instances create restored-vm \
    --source-machine-image=my-vm-image \
    --zone=us-central1-b
```

### Snapshots vs Images vs Machine Images

| Feature | Snapshots | Images | Machine Images |
|---------|-----------|--------|----------------|
| **Scope** | Single disk | Boot disk only | Entire VM |
| **Metadata** | Disk data only | OS + software | VM config + all disks |
| **Use Case** | Backup/restore | Template creation | Complete VM backup |
| **Cross-region** | Yes | Yes | Yes |
| **Boot capability** | No | Yes | Yes |
| **Cost** | Storage only | Storage only | Storage + metadata |

**DevOps Usage Patterns:**
```bash
# Snapshots - for data backup
gcloud compute disks snapshot data-disk --snapshot-name=data-backup

# Images - for golden templates
gcloud compute images create web-server-template \
    --source-disk=configured-boot-disk \
    --source-disk-zone=us-central1-a

# Machine Images - for complete DR
gcloud compute machine-images create production-backup \
    --source-instance=production-server
```

---

## 🔹 Persistent Disk Operations

### Mounting Persistent Disks on GCE VMs

**Attach and Mount New Disk:**
```bash
# 1. Create and attach disk
gcloud compute disks create data-disk \
    --size=100GB \
    --type=pd-ssd \
    --zone=us-central1-a

gcloud compute instances attach-disk my-instance \
    --disk=data-disk \
    --zone=us-central1-a

# 2. Format and mount (on the VM)
sudo lsblk  # Identify the new disk (e.g., /dev/sdb)

# Format with ext4
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

# Create mount point and mount
sudo mkdir -p /mnt/disks/data
sudo mount -o discard,defaults /dev/sdb /mnt/disks/data
sudo chmod a+w /mnt/disks/data

# Add to fstab for persistence
echo UUID=$(sudo blkid -s UUID -o value /dev/sdb) /mnt/disks/data ext4 discard,defaults,nofail 0 2 | sudo tee -a /etc/fstab
```

**Automated Disk Setup Script:**
```bash
#!/bin/bash
# automated-disk-setup.sh

DEVICE=$1
MOUNT_POINT=$2

if [ -z "$DEVICE" ] || [ -z "$MOUNT_POINT" ]; then
    echo "Usage: $0 <device> <mount_point>"
    echo "Example: $0 /dev/sdb /mnt/disks/data"
    exit 1
fi

# Check if device exists
if [ ! -b "$DEVICE" ]; then
    echo "Error: Device $DEVICE not found"
    exit 1
fi

# Format disk
echo "Formatting $DEVICE..."
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard $DEVICE

# Create mount point
echo "Creating mount point $MOUNT_POINT..."
sudo mkdir -p $MOUNT_POINT

# Mount disk
echo "Mounting $DEVICE to $MOUNT_POINT..."
sudo mount -o discard,defaults $DEVICE $MOUNT_POINT
sudo chmod a+w $MOUNT_POINT

# Add to fstab
UUID=$(sudo blkid -s UUID -o value $DEVICE)
echo "UUID=$UUID $MOUNT_POINT ext4 discard,defaults,nofail 0 2" | sudo tee -a /etc/fstab

echo "Disk setup complete!"
df -h $MOUNT_POINT
```

### Resizing Persistent Disks

**Online Disk Resize:**
```bash
# 1. Resize the disk (can be done while attached)
gcloud compute disks resize data-disk \
    --size=200GB \
    --zone=us-central1-a

# 2. Resize the file system (on the VM)
# For ext4 file systems
sudo resize2fs /dev/sdb

# For XFS file systems
sudo xfs_growfs /mnt/disks/data

# Verify the resize
df -h /mnt/disks/data
```

**Automated Resize Script:**
```bash
#!/bin/bash
# resize-disk.sh

DISK_NAME=$1
NEW_SIZE=$2
ZONE=$3
DEVICE=$4

# Resize the persistent disk
echo "Resizing disk $DISK_NAME to $NEW_SIZE..."
gcloud compute disks resize $DISK_NAME \
    --size=$NEW_SIZE \
    --zone=$ZONE

# Wait for resize to complete
sleep 10

# Resize the file system
echo "Resizing file system on $DEVICE..."
FS_TYPE=$(sudo blkid -o value -s TYPE $DEVICE)

case $FS_TYPE in
    ext4)
        sudo resize2fs $DEVICE
        ;;
    xfs)
        sudo xfs_growfs $(findmnt -n -o TARGET $DEVICE)
        ;;
    *)
        echo "Unsupported file system: $FS_TYPE"
        exit 1
        ;;
esac

echo "Resize complete!"
df -h $(findmnt -n -o TARGET $DEVICE)
```

### Real-World Scenarios: Persistent Disks

**Scenario 1: Database Migration**
```bash
#!/bin/bash
# Database migration with minimal downtime

# 1. Create snapshot of current database disk
gcloud compute disks snapshot db-disk \
    --snapshot-name=migration-backup \
    --zone=us-central1-a

# 2. Create new larger disk from snapshot
gcloud compute disks create db-disk-new \
    --source-snapshot=migration-backup \
    --size=1TB \
    --type=pd-ssd \
    --zone=us-central1-a

# 3. Stop database service
sudo systemctl stop postgresql

# 4. Detach old disk and attach new disk
gcloud compute instances detach-disk db-server \
    --disk=db-disk \
    --zone=us-central1-a

gcloud compute instances attach-disk db-server \
    --disk=db-disk-new \
    --zone=us-central1-a

# 5. Mount new disk and start service
sudo mount /dev/sdb /var/lib/postgresql
sudo systemctl start postgresql
```

**Scenario 2: Cross-Zone Migration**
```bash
#!/bin/bash
# Migrate VM with persistent disk to different zone

SOURCE_ZONE="us-central1-a"
TARGET_ZONE="us-central1-b"
INSTANCE_NAME="web-server"

# 1. Stop the instance
gcloud compute instances stop $INSTANCE_NAME --zone=$SOURCE_ZONE

# 2. Create machine image
gcloud compute machine-images create ${INSTANCE_NAME}-migration \
    --source-instance=$INSTANCE_NAME \
    --source-instance-zone=$SOURCE_ZONE

# 3. Create new instance in target zone
gcloud compute instances create ${INSTANCE_NAME}-new \
    --source-machine-image=${INSTANCE_NAME}-migration \
    --zone=$TARGET_ZONE

# 4. Update DNS/load balancer to point to new instance
# 5. Delete old instance after verification
```

**Scenario 3: Disaster Recovery**
```bash
#!/bin/bash
# Automated disaster recovery setup

# Create cross-region snapshot for DR
gcloud compute disks snapshot production-db \
    --snapshot-name=dr-backup-$(date +%Y%m%d) \
    --zone=us-central1-a \
    --storage-location=us-east1

# Create DR instance in different region
gcloud compute instances create dr-instance \
    --zone=us-east1-a \
    --machine-type=n1-standard-4 \
    --create-disk=name=dr-disk,source-snapshot=dr-backup-$(date +%Y%m%d),type=pd-ssd,size=500GB \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud
```

---

## 🔹 File Storage

### Introduction to Filestore

**Filestore Overview:**
- **Fully managed NFS** file system service
- **POSIX-compliant** shared storage
- **High performance** with consistent low latency
- **Scalable** from 1TB to 100TB+

**Filestore Tiers:**

| Tier | Performance | Capacity | Use Case |
|------|-------------|----------|----------|
| **Basic HDD** | 180 IOPS/TB | 1TB - 63.9TB | File shares, content repositories |
| **Basic SSD** | 480 IOPS/TB | 2.5TB - 63.9TB | High-performance workloads |
| **High Scale SSD** | 1,000 IOPS/TB | 10TB - 100TB | Enterprise applications |
| **Enterprise** | 1,500 IOPS/TB | 1TB - 10TB | Mission-critical workloads |

### Filestore Implementation

**Create Filestore Instance:**
```bash
# Basic Filestore instance
gcloud filestore instances create nfs-server \
    --zone=us-central1-a \
    --tier=STANDARD \
    --file-share=name="share1",capacity=1TB \
    --network=name="default"

# High-performance Filestore
gcloud filestore instances create high-perf-nfs \
    --zone=us-central1-a \
    --tier=PREMIUM \
    --file-share=name="data",capacity=5TB \
    --network=name="production-vpc" \
    --reserved-ip-range="10.0.0.0/29"
```

**Mount Filestore on Clients:**
```bash
# Install NFS client
sudo apt-get update
sudo apt-get install nfs-common

# Create mount point
sudo mkdir -p /mnt/nfs/share1

# Mount Filestore
sudo mount -t nfs -o vers=3 10.0.0.2:/share1 /mnt/nfs/share1

# Add to fstab for persistence
echo "10.0.0.2:/share1 /mnt/nfs/share1 nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

**Automated Filestore Setup:**
```bash
#!/bin/bash
# setup-filestore.sh

INSTANCE_NAME=$1
ZONE=$2
CAPACITY=$3
NETWORK=$4

# Create Filestore instance
echo "Creating Filestore instance: $INSTANCE_NAME"
gcloud filestore instances create $INSTANCE_NAME \
    --zone=$ZONE \
    --tier=STANDARD \
    --file-share=name="data",capacity=$CAPACITY \
    --network=name="$NETWORK"

# Wait for instance to be ready
echo "Waiting for Filestore instance to be ready..."
while true; do
    STATUS=$(gcloud filestore instances describe $INSTANCE_NAME \
        --zone=$ZONE \
        --format="value(state)")
    
    if [ "$STATUS" = "READY" ]; then
        break
    fi
    
    echo "Status: $STATUS, waiting..."
    sleep 30
done

# Get IP address
IP_ADDRESS=$(gcloud filestore instances describe $INSTANCE_NAME \
    --zone=$ZONE \
    --format="value(networks[0].ipAddresses[0])")

echo "Filestore instance ready at IP: $IP_ADDRESS"
echo "Mount command: sudo mount -t nfs -o vers=3 $IP_ADDRESS:/data /mnt/nfs"
```

### Real-World Scenarios: Block and File Storage

**Scenario 1: Shared Development Environment**
```bash
#!/bin/bash
# Setup shared development environment with Filestore

# 1. Create Filestore for shared code and data
gcloud filestore instances create dev-shared-storage \
    --zone=us-central1-a \
    --tier=STANDARD \
    --file-share=name="shared",capacity=2TB \
    --network=name="dev-vpc"

# 2. Create development VMs with shared storage
for i in {1..5}; do
    gcloud compute instances create dev-vm-$i \
        --zone=us-central1-a \
        --machine-type=e2-standard-4 \
        --image-family=ubuntu-2004-lts \
        --image-project=ubuntu-os-cloud \
        --network=dev-vpc \
        --metadata=startup-script='#!/bin/bash
            apt-get update
            apt-get install -y nfs-common
            mkdir -p /mnt/shared
            mount -t nfs -o vers=3 FILESTORE_IP:/shared /mnt/shared
            echo "FILESTORE_IP:/shared /mnt/shared nfs defaults,_netdev 0 0" >> /etc/fstab'
done
```

**Scenario 2: High-Performance Database Setup**
```bash
#!/bin/bash
# Database setup with optimal storage configuration

# 1. Create high-performance disks for database
gcloud compute disks create db-data-disk \
    --size=500GB \
    --type=pd-extreme \
    --provisioned-iops=50000 \
    --zone=us-central1-a

gcloud compute disks create db-log-disk \
    --size=100GB \
    --type=pd-ssd \
    --zone=us-central1-a

# 2. Create database VM with multiple disks
gcloud compute instances create database-server \
    --zone=us-central1-a \
    --machine-type=n1-highmem-8 \
    --boot-disk-size=50GB \
    --boot-disk-type=pd-ssd \
    --disk=name=db-data-disk,auto-delete=no \
    --disk=name=db-log-disk,auto-delete=no \
    --local-ssd=interface=SCSI \
    --local-ssd=interface=SCSI

# 3. Setup storage on the VM
gcloud compute ssh database-server --zone=us-central1-a --command='
    # Setup data disk
    sudo mkfs.ext4 -F /dev/sdb
    sudo mkdir -p /var/lib/postgresql/data
    sudo mount /dev/sdb /var/lib/postgresql/data
    
    # Setup log disk
    sudo mkfs.ext4 -F /dev/sdc
    sudo mkdir -p /var/lib/postgresql/logs
    sudo mount /dev/sdc /var/lib/postgresql/logs
    
    # Setup Local SSD RAID for temp space
    sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdd /dev/sde
    sudo mkfs.ext4 -F /dev/md0
    sudo mkdir -p /tmp/postgresql
    sudo mount /dev/md0 /tmp/postgresql
    
    # Update fstab
    echo "/dev/sdb /var/lib/postgresql/data ext4 defaults 0 2" | sudo tee -a /etc/fstab
    echo "/dev/sdc /var/lib/postgresql/logs ext4 defaults 0 2" | sudo tee -a /etc/fstab
    echo "/dev/md0 /tmp/postgresql ext4 defaults 0 2" | sudo tee -a /etc/fstab
'
```

**Scenario 3: Content Management System**
```bash
#!/bin/bash
# CMS setup with mixed storage types

# 1. Create Filestore for shared media files
gcloud filestore instances create cms-media-storage \
    --zone=us-central1-a \
    --tier=STANDARD \
    --file-share=name="media",capacity=5TB \
    --network=name="production-vpc"

# 2. Create web servers with local storage for cache
for i in {1..3}; do
    gcloud compute instances create web-server-$i \
        --zone=us-central1-a \
        --machine-type=n1-standard-4 \
        --boot-disk-size=50GB \
        --boot-disk-type=pd-ssd \
        --create-disk=name=cache-disk-$i,size=100GB,type=pd-ssd \
        --local-ssd=interface=SCSI \
        --network=production-vpc \
        --tags=web-server
done

# 3. Setup load balancer and configure storage
# Configure each web server to mount shared media storage
# Use local SSD for application cache
# Use persistent disk for application data
```

---

## 🔹 DevOps Best Practices

### Storage Performance Optimization

**Disk Performance Tuning:**
```bash
#!/bin/bash
# optimize-disk-performance.sh

DEVICE=$1

# Optimize I/O scheduler for SSD
echo noop | sudo tee /sys/block/${DEVICE#/dev/}/queue/scheduler

# Optimize read-ahead
sudo blockdev --setra 256 $DEVICE

# Mount with optimized options
sudo mount -o noatime,discard,barrier=0 $DEVICE /mnt/optimized

# Add optimized mount options to fstab
echo "$DEVICE /mnt/optimized ext4 noatime,discard,barrier=0,nofail 0 2" | sudo tee -a /etc/fstab
```

**Storage Monitoring Script:**
```python
#!/usr/bin/env python3
# storage-monitor.py

import subprocess
import json
import time

def get_disk_metrics():
    """Get disk performance metrics."""
    
    # Get disk usage
    df_output = subprocess.check_output(['df', '-h']).decode()
    
    # Get I/O statistics
    iostat_output = subprocess.check_output(['iostat', '-x', '1', '1']).decode()
    
    # Parse and return metrics
    metrics = {
        'timestamp': time.time(),
        'disk_usage': parse_df_output(df_output),
        'io_stats': parse_iostat_output(iostat_output)
    }
    
    return metrics

def check_disk_health():
    """Check disk health and performance."""
    
    metrics = get_disk_metrics()
    
    # Check for high utilization
    for mount, usage in metrics['disk_usage'].items():
        if usage['use_percent'] > 90:
            print(f"WARNING: {mount} is {usage['use_percent']}% full")
    
    # Check for high I/O wait
    for device, stats in metrics['io_stats'].items():
        if stats['util'] > 80:
            print(f"WARNING: {device} utilization is {stats['util']}%")

if __name__ == "__main__":
    check_disk_health()
```

### Backup and Disaster Recovery

**Automated Backup Strategy:**
```bash
#!/bin/bash
# comprehensive-backup.sh

PROJECT_ID="my-project"
BACKUP_RETENTION_DAYS=30

# Function to create snapshots with retention
create_snapshot_with_cleanup() {
    local disk_name=$1
    local zone=$2
    local description=$3
    
    # Create snapshot
    local snapshot_name="${disk_name}-$(date +%Y%m%d-%H%M%S)"
    
    gcloud compute disks snapshot $disk_name \
        --snapshot-name=$snapshot_name \
        --zone=$zone \
        --description="$description" \
        --labels=backup-type=automated,created-by=script
    
    # Cleanup old snapshots
    local cutoff_date=$(date -d "$BACKUP_RETENTION_DAYS days ago" +%Y%m%d)
    
    gcloud compute snapshots list \
        --filter="labels.backup-type=automated AND creationTimestamp<$cutoff_date" \
        --format="value(name)" | \
    while read snapshot; do
        echo "Deleting old snapshot: $snapshot"
        gcloud compute snapshots delete $snapshot --quiet
    done
}

# Backup all production disks
gcloud compute disks list \
    --filter="labels.environment=production" \
    --format="csv[no-heading](name,zone)" | \
while IFS=, read disk_name zone; do
    echo "Backing up $disk_name in $zone"
    create_snapshot_with_cleanup $disk_name $zone "Automated production backup"
done
```

### Infrastructure as Code

**Terraform Storage Configuration:**
```hcl
# storage.tf
# Persistent Disk Configuration
resource "google_compute_disk" "app_data" {
  name  = "app-data-disk"
  type  = "pd-ssd"
  zone  = var.zone
  size  = 100
  
  labels = {
    environment = "production"
    application = "web-app"
  }
  
  lifecycle {
    prevent_destroy = true
  }
}

# Snapshot Schedule
resource "google_compute_resource_policy" "daily_backup" {
  name   = "daily-backup-policy"
  region = var.region
  
  snapshot_schedule_policy {
    schedule {
      daily_schedule {
        days_in_cycle = 1
        start_time    = "04:00"
      }
    }
    
    retention_policy {
      max_retention_days    = 30
      on_source_disk_delete = "KEEP_AUTO_SNAPSHOTS"
    }
    
    snapshot_properties {
      labels = {
        backup_type = "automated"
        created_by  = "terraform"
      }
      storage_locations = [var.region]
      guest_flush       = true
    }
  }
}

# Attach policy to disk
resource "google_compute_disk_resource_policy_attachment" "backup_attachment" {
  name = google_compute_resource_policy.daily_backup.name
  disk = google_compute_disk.app_data.name
  zone = var.zone
}

# Filestore Instance
resource "google_filestore_instance" "shared_storage" {
  name     = "shared-nfs"
  location = var.zone
  tier     = "STANDARD"
  
  file_shares {
    capacity_gb = 1024
    name        = "shared_data"
  }
  
  networks {
    network = "default"
    modes   = ["MODE_IPV4"]
  }
  
  labels = {
    environment = "production"
    purpose     = "shared-storage"
  }
}
```

---

## 🔹 Monitoring & Troubleshooting

### Storage Monitoring

**Cloud Monitoring Metrics:**
```yaml
# Key storage metrics to monitor:
Disk Metrics:
  - compute.googleapis.com/instance/disk/read_bytes_count
  - compute.googleapis.com/instance/disk/write_bytes_count
  - compute.googleapis.com/instance/disk/read_ops_count
  - compute.googleapis.com/instance/disk/write_ops_count

Filestore Metrics:
  - file.googleapis.com/nfs/server/used_bytes
  - file.googleapis.com/nfs/server/free_bytes
  - file.googleapis.com/nfs/server/read_bytes_count
  - file.googleapis.com/nfs/server/write_bytes_count
```

**Custom Monitoring Dashboard:**
```python
#!/usr/bin/env python3
# create-storage-dashboard.py

from google.cloud import monitoring_dashboard_v1

def create_storage_dashboard(project_id):
    """Create comprehensive storage monitoring dashboard."""
    
    client = monitoring_dashboard_v1.DashboardsServiceClient()
    project_name = f"projects/{project_id}"
    
    dashboard = monitoring_dashboard_v1.Dashboard(
        display_name="Storage Performance Dashboard",
        grid_layout=monitoring_dashboard_v1.GridLayout(
            widgets=[
                # Disk IOPS widget
                monitoring_dashboard_v1.Widget(
                    title="Disk IOPS",
                    xy_chart=monitoring_dashboard_v1.XyChart(
                        data_sets=[
                            monitoring_dashboard_v1.DataSet(
                                time_series_query=monitoring_dashboard_v1.TimeSeriesQuery(
                                    time_series_filter=monitoring_dashboard_v1.TimeSeriesFilter(
                                        filter='metric.type="compute.googleapis.com/instance/disk/read_ops_count"',
                                        aggregation=monitoring_dashboard_v1.Aggregation(
                                            alignment_period={"seconds": 60},
                                            per_series_aligner=monitoring_dashboard_v1.Aggregation.Aligner.ALIGN_RATE
                                        )
                                    )
                                )
                            )
                        ]
                    )
                ),
                # Disk throughput widget
                monitoring_dashboard_v1.Widget(
                    title="Disk Throughput",
                    xy_chart=monitoring_dashboard_v1.XyChart(
                        data_sets=[
                            monitoring_dashboard_v1.DataSet(
                                time_series_query=monitoring_dashboard_v1.TimeSeriesQuery(
                                    time_series_filter=monitoring_dashboard_v1.TimeSeriesFilter(
                                        filter='metric.type="compute.googleapis.com/instance/disk/read_bytes_count"',
                                        aggregation=monitoring_dashboard_v1.Aggregation(
                                            alignment_period={"seconds": 60},
                                            per_series_aligner=monitoring_dashboard_v1.Aggregation.Aligner.ALIGN_RATE
                                        )
                                    )
                                )
                            )
                        ]
                    )
                )
            ]
        )
    )
    
    response = client.create_dashboard(
        parent=project_name,
        dashboard=dashboard
    )
    
    print(f"Created dashboard: {response.name}")

if __name__ == "__main__":
    create_storage_dashboard("my-project-id")
```

### Troubleshooting Common Issues

**Disk Performance Issues:**
```bash
#!/bin/bash
# diagnose-disk-performance.sh

DEVICE=$1

echo "Diagnosing performance for $DEVICE"

# Check current I/O statistics
echo "=== Current I/O Stats ==="
iostat -x 1 3 $DEVICE

# Check disk utilization
echo "=== Disk Utilization ==="
df -h $DEVICE

# Check for I/O errors
echo "=== I/O Errors ==="
dmesg | grep -i "error\|fail" | grep $DEVICE

# Check mount options
echo "=== Mount Options ==="
mount | grep $DEVICE

# Check file system errors
echo "=== File System Check ==="
sudo fsck -n $DEVICE

# Performance recommendations
echo "=== Performance Recommendations ==="
if [[ $DEVICE == *"ssd"* ]]; then
    echo "- Use noop or deadline I/O scheduler for SSDs"
    echo "- Mount with noatime and discard options"
    echo "- Consider RAID 0 for multiple Local SSDs"
else
    echo "- Use cfq I/O scheduler for HDDs"
    echo "- Optimize read-ahead settings"
    echo "- Consider upgrading to SSD for better performance"
fi
```

**Filestore Connectivity Issues:**
```bash
#!/bin/bash
# diagnose-filestore.sh

FILESTORE_IP=$1
SHARE_NAME=$2

echo "Diagnosing Filestore connectivity to $FILESTORE_IP:/$SHARE_NAME"

# Check network connectivity
echo "=== Network Connectivity ==="
ping -c 3 $FILESTORE_IP

# Check NFS service
echo "=== NFS Service Check ==="
rpcinfo -p $FILESTORE_IP

# Check available shares
echo "=== Available Shares ==="
showmount -e $FILESTORE_IP

# Test mount
echo "=== Test Mount ==="
sudo mkdir -p /tmp/test-mount
sudo mount -t nfs -o vers=3 $FILESTORE_IP:/$SHARE_NAME /tmp/test-mount

if [ $? -eq 0 ]; then
    echo "Mount successful"
    ls -la /tmp/test-mount
    sudo umount /tmp/test-mount
else
    echo "Mount failed - check firewall rules and VPC connectivity"
fi

# Check firewall rules
echo "=== Firewall Rules ==="
gcloud compute firewall-rules list --filter="direction=INGRESS AND allowed.ports:2049"
```

This comprehensive GCP Storage guide provides you with the knowledge and practical examples needed to master Google Cloud storage services from a DevOps perspective. The guide covers everything from basic concepts to advanced implementation patterns, monitoring, and troubleshooting techniques.
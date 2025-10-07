# 🚀 Kubernetes Jobs & CronJobs - DevOps Interview Guide

## Table of Contents
1. [Jobs Overview](#jobs-overview)
2. [Job Types & Patterns](#job-types--patterns)
3. [Job Manifest Attributes](#job-manifest-attributes)
4. [CronJobs Overview](#cronjobs-overview)
5. [CronJob Manifest Attributes](#cronjob-manifest-attributes)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Interview Questions](#interview-questions)

---

## 🔹 Jobs Overview

### What are Kubernetes Jobs?

**Jobs** run pods to completion for batch processing, one-time tasks, or finite workloads.

**Key Characteristics:**
- Pods run until successful completion
- Automatically restart failed pods (based on restartPolicy)
- Track completion status
- Clean up completed pods (configurable)

### When to Use Jobs

```yaml
# Use Cases:
- Database migrations
- Data processing tasks
- Backup operations
- Batch computations
- One-time setup tasks
- ETL processes
```

---

## 🔹 Job Types & Patterns

### 1. Single Job (One Pod)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: single-task
spec:
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo 'Processing data...' && sleep 30"]
      restartPolicy: Never
```

### 2. Parallel Jobs (Fixed Completion Count)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 5        # Total successful completions needed
  parallelism: 2        # Max pods running simultaneously
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo 'Task $HOSTNAME completed'"]
      restartPolicy: Never
```

### 3. Work Queue Pattern
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: work-queue
spec:
  parallelism: 3        # Multiple workers
  # No completions specified - workers pull from queue
  template:
    spec:
      containers:
      - name: worker
        image: my-app:latest
        env:
        - name: QUEUE_URL
          value: "redis://redis-service:6379"
      restartPolicy: Never
```

---

## 🔹 Job Manifest Attributes

### Core Spec Fields

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: example-job
  namespace: default
  labels:
    app: batch-processor
    version: v1.0
  annotations:
    description: "Data processing job"
spec:
  # Completion Settings
  completions: 1                    # Total successful pods needed
  parallelism: 1                    # Max concurrent pods
  
  # Failure Handling
  backoffLimit: 6                   # Max retries before marking failed
  activeDeadlineSeconds: 3600       # Max runtime (1 hour)
  
  # Pod Management
  ttlSecondsAfterFinished: 100      # Auto-cleanup after completion
  
  # Pod Template
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      # Container Configuration
      containers:
      - name: processor
        image: my-processor:v1.0
        command: ["/bin/sh"]
        args: ["-c", "process-data.sh"]
        
        # Resource Management
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Environment Variables
        env:
        - name: BATCH_SIZE
          value: "1000"
        - name: DB_CONNECTION
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: connection-string
        
        # Volume Mounts
        volumeMounts:
        - name: data-volume
          mountPath: /data
        - name: config-volume
          mountPath: /config
      
      # Volumes
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: data-pvc
      - name: config-volume
        configMap:
          name: processor-config
      
      # Pod Settings
      restartPolicy: Never            # Never, OnFailure
      serviceAccountName: job-sa
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
```

### Advanced Job Configuration

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: advanced-job
spec:
  # Completion Mode (Kubernetes 1.21+)
  completionMode: Indexed           # Indexed or NonIndexed
  completions: 5
  parallelism: 2
  
  # Suspend Job
  suspend: false                    # Pause job execution
  
  # Pod Failure Policy (Kubernetes 1.25+)
  podFailurePolicy:
    rules:
    - action: FailJob              # FailJob, Ignore, Count
      onExitCodes:
        containerName: processor
        operator: In
        values: [1, 2, 3]
    - action: Ignore
      onPodConditions:
      - type: DisruptionTarget
  
  template:
    spec:
      containers:
      - name: processor
        image: processor:latest
        env:
        - name: JOB_COMPLETION_INDEX  # Available in Indexed mode
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
      restartPolicy: Never
```

---

## 🔹 CronJobs Overview

### What are CronJobs?

**CronJobs** create Jobs on a time-based schedule using cron syntax.

**Key Features:**
- Schedule recurring tasks
- Automatic Job creation
- Concurrency control
- History management
- Timezone support (Kubernetes 1.24+)

### Cron Schedule Syntax

```bash
# Format: "minute hour day-of-month month day-of-week"
# Examples:
"0 2 * * *"        # Daily at 2:00 AM
"*/15 * * * *"     # Every 15 minutes
"0 0 * * 0"        # Weekly on Sunday at midnight
"0 0 1 * *"        # Monthly on 1st day at midnight
"0 9-17 * * 1-5"   # Hourly during business hours (9-5, Mon-Fri)
"@hourly"          # Predefined: @yearly, @monthly, @weekly, @daily, @hourly
```

---

## 🔹 CronJob Manifest Attributes

### Complete CronJob Configuration

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cronjob
  namespace: production
  labels:
    app: backup-system
    component: database-backup
spec:
  # Schedule Configuration
  schedule: "0 2 * * *"                    # Daily at 2 AM
  timeZone: "America/New_York"             # Kubernetes 1.24+
  
  # Concurrency Control
  concurrencyPolicy: Forbid                # Allow, Forbid, Replace
  
  # Job Management
  suspend: false                           # Pause CronJob
  successfulJobsHistoryLimit: 3            # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1                # Keep last 1 failed job
  startingDeadlineSeconds: 300             # Max delay to start job
  
  # Job Template
  jobTemplate:
    metadata:
      labels:
        cronjob: backup-cronjob
    spec:
      # Job-specific settings
      activeDeadlineSeconds: 3600          # 1 hour timeout
      backoffLimit: 2                      # Max retries
      ttlSecondsAfterFinished: 86400       # Cleanup after 24 hours
      
      template:
        metadata:
          labels:
            app: backup-worker
        spec:
          containers:
          - name: backup
            image: postgres:13
            command:
            - /bin/bash
            - -c
            - |
              echo "Starting backup at $(date)"
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > /backup/backup-$(date +%Y%m%d-%H%M%S).sql
              echo "Backup completed at $(date)"
            
            env:
            - name: DB_HOST
              value: "postgres-service"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_NAME
              value: "production_db"
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
            
            resources:
              requests:
                memory: "256Mi"
                cpu: "100m"
              limits:
                memory: "1Gi"
                cpu: "500m"
          
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          
          restartPolicy: OnFailure
          serviceAccountName: backup-sa
```

### CronJob Concurrency Policies

```yaml
# 1. Allow (Default) - Multiple jobs can run concurrently
spec:
  concurrencyPolicy: Allow
  schedule: "*/5 * * * *"  # Every 5 minutes

# 2. Forbid - Skip new job if previous is still running
spec:
  concurrencyPolicy: Forbid
  schedule: "0 */2 * * *"  # Every 2 hours

# 3. Replace - Kill old job and start new one
spec:
  concurrencyPolicy: Replace
  schedule: "*/10 * * * *"  # Every 10 minutes
```

---

## 🔹 Practical Examples

### 1. Database Migration Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  labels:
    app: migration
    version: v2.1.0
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 1800  # 30 minutes
  template:
    spec:
      initContainers:
      - name: wait-for-db
        image: busybox:1.35
        command: ['sh', '-c']
        args:
        - |
          until nc -z postgres-service 5432; do
            echo "Waiting for database..."
            sleep 5
          done
          echo "Database is ready!"
      
      containers:
      - name: migrator
        image: migrate/migrate:v4.15.2
        command: ["/migrate"]
        args:
        - "-path=/migrations"
        - "-database=postgres://$(DB_USER):$(DB_PASS)@postgres-service:5432/$(DB_NAME)?sslmode=disable"
        - "up"
        
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: DB_NAME
          value: "production"
        
        volumeMounts:
        - name: migrations
          mountPath: /migrations
          readOnly: true
      
      volumes:
      - name: migrations
        configMap:
          name: migration-scripts
      
      restartPolicy: Never
```

### 2. Log Cleanup CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
spec:
  schedule: "0 1 * * *"  # Daily at 1 AM
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: alpine:3.18
            command:
            - /bin/sh
            - -c
            - |
              echo "Starting log cleanup..."
              
              # Remove logs older than 30 days
              find /var/log -name "*.log" -type f -mtime +30 -delete
              
              # Compress logs older than 7 days
              find /var/log -name "*.log" -type f -mtime +7 -exec gzip {} \;
              
              # Clean up empty directories
              find /var/log -type d -empty -delete
              
              echo "Log cleanup completed"
            
            volumeMounts:
            - name: log-volume
              mountPath: /var/log
            
            resources:
              requests:
                memory: "64Mi"
                cpu: "50m"
              limits:
                memory: "128Mi"
                cpu: "100m"
          
          volumes:
          - name: log-volume
            hostPath:
              path: /var/log
              type: Directory
          
          restartPolicy: OnFailure
          nodeSelector:
            node-type: worker
```

### 3. Data Processing Pipeline

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-pipeline
spec:
  completions: 10
  parallelism: 3
  completionMode: Indexed
  
  template:
    spec:
      containers:
      - name: processor
        image: python:3.11-slim
        command: ["python", "/app/process.py"]
        
        env:
        - name: WORKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
        - name: TOTAL_WORKERS
          value: "10"
        - name: S3_BUCKET
          value: "data-processing-bucket"
        
        volumeMounts:
        - name: app-code
          mountPath: /app
        - name: temp-storage
          mountPath: /tmp/processing
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      
      volumes:
      - name: app-code
        configMap:
          name: processing-scripts
      - name: temp-storage
        emptyDir:
          sizeLimit: "2Gi"
      
      restartPolicy: Never
```

---

## 🔹 Best Practices

### 1. Resource Management

```yaml
# Always set resource requests and limits
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Use appropriate QoS classes
# Guaranteed: requests = limits
# Burstable: requests < limits
# BestEffort: no requests/limits (avoid for jobs)
```

### 2. Error Handling

```yaml
spec:
  # Limit retries to prevent infinite loops
  backoffLimit: 3
  
  # Set reasonable timeouts
  activeDeadlineSeconds: 3600
  
  # Use appropriate restart policy
  template:
    spec:
      restartPolicy: Never  # or OnFailure
```

### 3. Cleanup Strategy

```yaml
# Automatic cleanup
spec:
  ttlSecondsAfterFinished: 86400  # 24 hours

# For CronJobs
spec:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

### 4. Security Best Practices

```yaml
spec:
  template:
    spec:
      # Use dedicated service account
      serviceAccountName: job-service-account
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: worker
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

### 5. Monitoring and Observability

```yaml
# Add labels for monitoring
metadata:
  labels:
    app: data-processor
    component: etl
    version: v1.2.0
    environment: production

# Use annotations for metadata
annotations:
  description: "Daily ETL processing job"
  owner: "data-team@company.com"
  runbook: "https://wiki.company.com/etl-runbook"
```

---

## 🔹 Troubleshooting

### Common Issues and Solutions

#### 1. Job Never Completes

```bash
# Check job status
kubectl describe job my-job

# Check pod logs
kubectl logs -l job-name=my-job

# Common causes:
# - Incorrect restartPolicy (should be Never or OnFailure)
# - Application doesn't exit properly
# - Infinite loops in application code
```

#### 2. CronJob Not Triggering

```bash
# Check CronJob status
kubectl describe cronjob my-cronjob

# Check schedule syntax
kubectl get cronjob my-cronjob -o yaml | grep schedule

# Common issues:
# - Invalid cron syntax
# - Timezone issues
# - CronJob suspended (suspend: true)
```

#### 3. Resource Issues

```bash
# Check resource usage
kubectl top pods -l job-name=my-job

# Check events
kubectl get events --field-selector involvedObject.name=my-job

# Solutions:
# - Increase resource limits
# - Add node selectors
# - Use resource quotas
```

### Debugging Commands

```bash
# List all jobs
kubectl get jobs

# Get job details
kubectl describe job <job-name>

# View job logs
kubectl logs job/<job-name>

# List CronJobs
kubectl get cronjobs

# Check CronJob history
kubectl get jobs -l cronjob=<cronjob-name>

# Manual CronJob trigger
kubectl create job --from=cronjob/<cronjob-name> <job-name>

# Delete completed jobs
kubectl delete jobs --field-selector status.successful=1
```

---

## 🔹 Interview Questions

### Technical Questions

**Q1: What's the difference between a Job and a Deployment?**
- **Job**: Runs pods to completion, for batch/finite tasks
- **Deployment**: Maintains desired number of running pods continuously

**Q2: Explain CronJob concurrency policies.**
- **Allow**: Multiple jobs can run simultaneously (default)
- **Forbid**: Skip new job if previous is still running
- **Replace**: Terminate old job and start new one

**Q3: How do you handle failed Jobs?**
```yaml
spec:
  backoffLimit: 3              # Max retries
  activeDeadlineSeconds: 3600  # Timeout
  template:
    spec:
      restartPolicy: OnFailure # Restart failed containers
```

**Q4: What happens if a CronJob schedule is missed?**
- If `startingDeadlineSeconds` is set, job won't start if deadline passed
- Without deadline, job starts as soon as possible
- Use `concurrencyPolicy: Forbid` to prevent overlapping jobs

### Scenario-Based Questions

**Q5: Design a backup strategy using CronJobs.**
```yaml
# Multiple CronJobs for different backup types
- Daily incremental: "0 2 * * *"
- Weekly full: "0 1 * * 0"
- Monthly archive: "0 0 1 * *"
```

**Q6: How would you process a large dataset in parallel?**
```yaml
spec:
  completions: 100    # Total chunks
  parallelism: 10     # Concurrent workers
  completionMode: Indexed  # Each worker gets unique index
```

**Q7: Troubleshoot a Job that keeps failing.**
```bash
# Check job status and events
kubectl describe job failing-job

# Check pod logs
kubectl logs -l job-name=failing-job --previous

# Common fixes:
# - Increase backoffLimit
# - Add resource limits
# - Fix application exit codes
# - Check dependencies (databases, services)
```

### Best Practices Questions

**Q8: How do you ensure Jobs don't consume too many resources?**
- Set resource requests and limits
- Use ResourceQuotas and LimitRanges
- Implement proper cleanup with `ttlSecondsAfterFinished`
- Monitor resource usage

**Q9: Security considerations for Jobs?**
- Use dedicated ServiceAccounts with minimal permissions
- Run as non-root user
- Use read-only root filesystem
- Scan container images for vulnerabilities
- Use network policies to restrict communication

**Q10: How do you monitor Job execution?**
- Use Prometheus metrics (kube-state-metrics)
- Set up alerts for failed jobs
- Log aggregation for job outputs
- Use labels for proper categorization
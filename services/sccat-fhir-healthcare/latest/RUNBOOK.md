# Day 2 Operations Runbook

## Service: sccat-fhir-healthcare

**Version**: v1
**Last Updated**: 2026-04-26

---

## Table of Contents

1. [Pre-Deployment Checklist](#pre-deployment-checklist)
2. [Deployment Procedures](#deployment-procedures)
3. [Health Checks](#health-checks)
4. [Monitoring & Alerts](#monitoring--alerts)
5. [Backup & Recovery](#backup--recovery)
6. [Scaling Operations](#scaling-operations)
7. [Troubleshooting](#troubleshooting)
8. [Maintenance Windows](#maintenance-windows)
9. [Security Operations](#security-operations)
10. [Incident Response](#incident-response)

---

## Pre-Deployment Checklist

### 1. Namespace Preparation

```bash
# Create namespace
kubectl create namespace fhir-healthcare

# Verify namespace
kubectl get namespace fhir-healthcare
```

### 2. Secret Creation

#### FHIR Database Secret (Required)

```bash
kubectl create secret generic fhir-db-credentials \
  --namespace=fhir-healthcare \
  --from-literal=postgres-db=fhirdb \
  --from-literal=postgres-user=fhiruser \
  --from-literal=postgres-password='CHANGE-ME-STRONG-PASSWORD'

# Verify secret
kubectl get secret fhir-db-credentials -n fhir-healthcare
```

#### Keycloak Secret (Optional)

```bash
kubectl create secret generic keycloak-credentials \
  --namespace=fhir-healthcare \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='CHANGE-ME-STRONG-PASSWORD'
```

#### webMethods Secret (Optional)

```bash
kubectl create secret generic webmethods-credentials \
  --namespace=fhir-healthcare \
  --from-literal=api-key='YOUR-API-KEY' \
  --from-literal=claim-endpoint='https://prod121185.a-vir-r1.int.ipaas.automation.ibm.com/...' \
  --from-literal=claimresponse-endpoint='https://prod121185.a-vir-r1.int.ipaas.automation.ibm.com/...'
```

### 3. Resource Verification

```bash
# Check cluster resources
kubectl top nodes

# Verify storage class
kubectl get storageclass

# Check available resources
kubectl describe nodes | grep -A 5 "Allocated resources"
```

### 4. Pre-Deployment Validation

```bash
# Validate manifests
kubectl kustomize services/sccat-fhir-healthcare/latest/manifests

# Dry-run deployment
kubectl apply -k instances/test-fhir-instance/ --dry-run=client
```

---

## Deployment Procedures

### Standard Deployment

```bash
# Deploy via ArgoCD (GitOps)
# ArgoCD automatically detects changes in Git and applies them

# Manual deployment (if needed)
kubectl apply -k instances/test-fhir-instance/

# Watch deployment progress
kubectl get pods -n fhir-healthcare -w
```

### Deployment Order

1. **PostgreSQL StatefulSet** (30-60 seconds)
2. **FHIR Server Deployment** (60-120 seconds)
3. **Dashboard Deployment** (30-60 seconds)

### Expected Timeline

| Phase | Duration | Status Check |
|-------|----------|--------------|
| PostgreSQL Init | 30-60s | `kubectl get statefulset -n fhir-healthcare` |
| FHIR Server Start | 60-120s | `kubectl get deployment -n fhir-healthcare` |
| Dashboard Start | 30-60s | `kubectl get deployment -n fhir-healthcare` |
| **Total** | **2-4 minutes** | All pods Running |

---

## Health Checks

### Automated Health Checks

All components have built-in Kubernetes health checks:

```yaml
# Readiness Probe - Service ready to accept traffic
readinessProbe:
  httpGet:
    path: /fhir/metadata
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10

# Liveness Probe - Service is alive
livenessProbe:
  httpGet:
    path: /fhir/metadata
    port: 8080
  initialDelaySeconds: 120
  periodSeconds: 30
```

### Manual Health Checks

#### PostgreSQL Health

```bash
# Check PostgreSQL pod
kubectl get pods -n fhir-healthcare -l app=postgres

# Test database connection
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "SELECT version();"

# Check database size
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "SELECT pg_size_pretty(pg_database_size('fhirdb'));"
```

#### FHIR Server Health

```bash
# Check FHIR pods
kubectl get pods -n fhir-healthcare -l app=fhir

# Test FHIR metadata endpoint
kubectl port-forward svc/test-fhir-instance-fhir 8080:8080 -n fhir-healthcare &
curl http://localhost:8080/fhir/metadata | jq '.fhirVersion'

# Test patient search
curl http://localhost:8080/fhir/Patient | jq '.total'
```

#### Dashboard Health

```bash
# Check dashboard pods
kubectl get pods -n fhir-healthcare -l app=dashboard

# Test dashboard
kubectl port-forward svc/test-fhir-instance-dashboard 3000:3000 -n fhir-healthcare &
curl -I http://localhost:3000
```

### Health Check Script

```bash
#!/bin/bash
# health-check.sh

NAMESPACE="fhir-healthcare"
INSTANCE="test-fhir-instance"

echo "=== Health Check: $INSTANCE ==="

# Check all pods
echo -e "\n1. Pod Status:"
kubectl get pods -n $NAMESPACE

# Check PostgreSQL
echo -e "\n2. PostgreSQL Health:"
kubectl exec -it ${INSTANCE}-postgres-0 -n $NAMESPACE -- \
  psql -U fhiruser -d fhirdb -c "SELECT 1" 2>/dev/null && echo "✅ PostgreSQL OK" || echo "❌ PostgreSQL FAILED"

# Check FHIR Server
echo -e "\n3. FHIR Server Health:"
kubectl exec -it deployment/${INSTANCE}-fhir -n $NAMESPACE -- \
  curl -s http://localhost:8080/fhir/metadata > /dev/null && echo "✅ FHIR Server OK" || echo "❌ FHIR Server FAILED"

# Check Dashboard
echo -e "\n4. Dashboard Health:"
kubectl exec -it deployment/${INSTANCE}-dashboard -n $NAMESPACE -- \
  curl -s http://localhost:3000 > /dev/null && echo "✅ Dashboard OK" || echo "❌ Dashboard FAILED"

# Resource usage
echo -e "\n5. Resource Usage:"
kubectl top pods -n $NAMESPACE
```

---

## Monitoring & Alerts

### Key Metrics to Monitor

#### PostgreSQL Metrics

```bash
# Database connections
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "SELECT count(*) FROM pg_stat_activity;"

# Database size growth
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "SELECT pg_size_pretty(pg_database_size('fhirdb'));"

# Slow queries
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "SELECT query, calls, total_time FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;"
```

#### FHIR Server Metrics

```bash
# Request rate
kubectl logs deployment/test-fhir-instance-fhir -n fhir-healthcare --tail=100 | grep "HTTP"

# Error rate
kubectl logs deployment/test-fhir-instance-fhir -n fhir-healthcare --tail=100 | grep "ERROR"

# Response times
kubectl logs deployment/test-fhir-instance-fhir -n fhir-healthcare --tail=100 | grep "duration"
```

#### Resource Metrics

```bash
# CPU and Memory usage
kubectl top pods -n fhir-healthcare

# Persistent volume usage
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- df -h /var/lib/postgresql/data
```

### Alert Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Pod Restarts | > 3/hour | > 10/hour | Check logs, investigate crash |
| CPU Usage | > 70% | > 90% | Scale up replicas |
| Memory Usage | > 80% | > 95% | Scale up resources |
| Disk Usage | > 75% | > 90% | Expand PV or cleanup |
| Response Time | > 2s | > 5s | Check database, scale up |
| Error Rate | > 1% | > 5% | Check logs, investigate |

---

## Backup & Recovery

### PostgreSQL Backup

#### Manual Backup

```bash
# Create backup
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  pg_dump -U fhiruser fhirdb > fhir-backup-$(date +%Y%m%d-%H%M%S).sql

# Verify backup
ls -lh fhir-backup-*.sql
```

#### Automated Backup (CronJob)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: fhir-healthcare
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:16-alpine
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h test-fhir-instance-postgres -U fhiruser fhirdb | \
              gzip > /backup/fhir-backup-$(date +%Y%m%d-%H%M%S).sql.gz
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: fhir-db-credentials
                  key: postgres-password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
          restartPolicy: OnFailure
```

### Restore Procedures

#### Full Database Restore

```bash
# Stop FHIR server to prevent writes
kubectl scale deployment test-fhir-instance-fhir --replicas=0 -n fhir-healthcare

# Copy backup to pod
kubectl cp fhir-backup-20260426.sql \
  test-fhir-instance-postgres-0:/tmp/restore.sql -n fhir-healthcare

# Restore database
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -f /tmp/restore.sql

# Restart FHIR server
kubectl scale deployment test-fhir-instance-fhir --replicas=1 -n fhir-healthcare

# Verify restoration
kubectl logs deployment/test-fhir-instance-fhir -n fhir-healthcare --tail=50
```

#### Point-in-Time Recovery

```bash
# Requires WAL archiving enabled in PostgreSQL
# Configure in postgres-statefulset.yaml:
# - archive_mode = on
# - archive_command = 'cp %p /archive/%f'

# Restore to specific timestamp
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  pg_restore --recovery-target-time='2026-04-26 10:00:00'
```

### Backup Retention Policy

| Backup Type | Retention | Storage Location |
|-------------|-----------|------------------|
| Daily | 7 days | PVC (local) |
| Weekly | 4 weeks | Object storage |
| Monthly | 12 months | Object storage |
| Yearly | 7 years | Cold storage |

---

## Scaling Operations

### Horizontal Scaling

#### Scale FHIR Server

```bash
# Scale up
kubectl scale deployment test-fhir-instance-fhir --replicas=3 -n fhir-healthcare

# Scale down
kubectl scale deployment test-fhir-instance-fhir --replicas=1 -n fhir-healthcare

# Verify scaling
kubectl get pods -n fhir-healthcare -l app=fhir
```

#### Scale Dashboard

```bash
# Scale up
kubectl scale deployment test-fhir-instance-dashboard --replicas=5 -n fhir-healthcare

# Scale down
kubectl scale deployment test-fhir-instance-dashboard --replicas=2 -n fhir-healthcare
```

#### Auto-scaling (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fhir-hpa
  namespace: fhir-healthcare
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: test-fhir-instance-fhir
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Vertical Scaling

#### Increase Resource Limits

```bash
# Edit deployment
kubectl edit deployment test-fhir-instance-fhir -n fhir-healthcare

# Update resources:
resources:
  requests:
    memory: "2Gi"
    cpu: "1"
  limits:
    memory: "4Gi"
    cpu: "4"

# Restart pods to apply changes
kubectl rollout restart deployment/test-fhir-instance-fhir -n fhir-healthcare
```

#### Expand PostgreSQL Storage

```bash
# Edit PVC (if storage class supports expansion)
kubectl edit pvc postgres-data-test-fhir-instance-postgres-0 -n fhir-healthcare

# Update size:
spec:
  resources:
    requests:
      storage: 50Gi

# Verify expansion
kubectl get pvc -n fhir-healthcare
```

---

## Troubleshooting

### Common Issues

#### 1. Pod Not Starting

**Symptoms**: Pod stuck in `Pending` or `CrashLoopBackOff`

```bash
# Check pod status
kubectl describe pod <pod-name> -n fhir-healthcare

# Common causes:
# - Secret not found
# - Insufficient resources
# - Image pull error
# - Volume mount issues

# Check events
kubectl get events -n fhir-healthcare --sort-by='.lastTimestamp'
```

**Solutions**:

```bash
# Verify secrets exist
kubectl get secrets -n fhir-healthcare

# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check image availability
kubectl describe pod <pod-name> -n fhir-healthcare | grep "Image"
```

#### 2. Database Connection Failures

**Symptoms**: FHIR server logs show connection errors

```bash
# Check FHIR server logs
kubectl logs deployment/test-fhir-instance-fhir -n fhir-healthcare | grep "database"

# Test PostgreSQL connectivity
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "SELECT 1"
```

**Solutions**:

```bash
# Verify secret values
kubectl get secret fhir-db-credentials -n fhir-healthcare -o yaml

# Check PostgreSQL service
kubectl get svc test-fhir-instance-postgres -n fhir-healthcare

# Restart FHIR server
kubectl rollout restart deployment/test-fhir-instance-fhir -n fhir-healthcare
```

#### 3. High Memory Usage

**Symptoms**: Pods being OOMKilled

```bash
# Check memory usage
kubectl top pods -n fhir-healthcare

# Check OOM events
kubectl get events -n fhir-healthcare | grep "OOMKilled"
```

**Solutions**:

```bash
# Increase memory limits
kubectl set resources deployment test-fhir-instance-fhir \
  --limits=memory=4Gi -n fhir-healthcare

# Scale horizontally instead
kubectl scale deployment test-fhir-instance-fhir --replicas=3 -n fhir-healthcare
```

#### 4. Slow FHIR Queries

**Symptoms**: Response times > 5 seconds

```bash
# Check slow queries in PostgreSQL
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "SELECT query, calls, total_time FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;"
```

**Solutions**:

```bash
# Add database indexes (if needed)
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "CREATE INDEX idx_patient_identifier ON patient(identifier);"

# Increase database resources
kubectl edit statefulset test-fhir-instance-postgres -n fhir-healthcare
```

#### 5. Dashboard Not Loading

**Symptoms**: 502/504 errors from dashboard

```bash
# Check dashboard logs
kubectl logs deployment/test-fhir-instance-dashboard -n fhir-healthcare

# Check FHIR server connectivity
kubectl exec -it deployment/test-fhir-instance-dashboard -n fhir-healthcare -- \
  curl http://test-fhir-instance-fhir:8080/fhir/metadata
```

**Solutions**:

```bash
# Restart dashboard
kubectl rollout restart deployment/test-fhir-instance-dashboard -n fhir-healthcare

# Check service endpoints
kubectl get endpoints -n fhir-healthcare
```

### Debug Commands

```bash
# Get all resources
kubectl get all -n fhir-healthcare

# Describe all pods
kubectl describe pods -n fhir-healthcare

# View logs (last 100 lines)
kubectl logs deployment/test-fhir-instance-fhir -n fhir-healthcare --tail=100

# Follow logs in real-time
kubectl logs -f deployment/test-fhir-instance-fhir -n fhir-healthcare

# Execute shell in pod
kubectl exec -it deployment/test-fhir-instance-fhir -n fhir-healthcare -- /bin/bash

# Port forward for local testing
kubectl port-forward svc/test-fhir-instance-fhir 8080:8080 -n fhir-healthcare
```

---

## Maintenance Windows

### Planned Maintenance

#### Pre-Maintenance Checklist

- [ ] Notify users 48 hours in advance
- [ ] Create full database backup
- [ ] Verify backup integrity
- [ ] Document current state (pod count, versions)
- [ ] Prepare rollback plan

#### Maintenance Procedure

```bash
# 1. Create backup
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  pg_dump -U fhiruser fhirdb > pre-maintenance-backup.sql

# 2. Scale down to maintenance mode
kubectl scale deployment test-fhir-instance-fhir --replicas=0 -n fhir-healthcare
kubectl scale deployment test-fhir-instance-dashboard --replicas=0 -n fhir-healthcare

# 3. Perform maintenance (e.g., database migration)
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -f /tmp/migration.sql

# 4. Scale back up
kubectl scale deployment test-fhir-instance-fhir --replicas=1 -n fhir-healthcare
kubectl scale deployment test-fhir-instance-dashboard --replicas=2 -n fhir-healthcare

# 5. Verify health
./health-check.sh
```

#### Post-Maintenance Checklist

- [ ] Verify all pods are running
- [ ] Test FHIR API endpoints
- [ ] Test dashboard functionality
- [ ] Monitor logs for errors
- [ ] Notify users of completion

### Rolling Updates

```bash
# Update FHIR server image
kubectl set image deployment/test-fhir-instance-fhir \
  fhir=hapiproject/hapi:7.0.3 -n fhir-healthcare

# Monitor rollout
kubectl rollout status deployment/test-fhir-instance-fhir -n fhir-healthcare

# Rollback if needed
kubectl rollout undo deployment/test-fhir-instance-fhir -n fhir-healthcare
```

---

## Security Operations

### Secret Rotation

#### Rotate Database Password

```bash
# 1. Update secret
kubectl create secret generic fhir-db-credentials-new \
  --namespace=fhir-healthcare \
  --from-literal=postgres-db=fhirdb \
  --from-literal=postgres-user=fhiruser \
  --from-literal=postgres-password='NEW-STRONG-PASSWORD'

# 2. Update PostgreSQL password
kubectl exec -it test-fhir-instance-postgres-0 -n fhir-healthcare -- \
  psql -U fhiruser -d fhirdb -c "ALTER USER fhiruser WITH PASSWORD 'NEW-STRONG-PASSWORD';"

# 3. Update deployment to use new secret
kubectl patch deployment test-fhir-instance-fhir -n fhir-healthcare \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"fhir","envFrom":[{"secretRef":{"name":"fhir-db-credentials-new"}}]}]}}}}'

# 4. Delete old secret
kubectl delete secret fhir-db-credentials -n fhir-healthcare
```

### Security Scanning

```bash
# Scan images for vulnerabilities
trivy image hapiproject/hapi:latest
trivy image postgres:16-alpine
trivy image nginx:alpine

# Scan Kubernetes manifests
trivy config services/sccat-fhir-healthcare/latest/manifests/
```

### Access Control

```bash
# Create read-only role
kubectl create role fhir-reader \
  --verb=get,list,watch \
  --resource=pods,services,deployments \
  -n fhir-healthcare

# Bind role to user
kubectl create rolebinding fhir-reader-binding \
  --role=fhir-reader \
  --user=john@example.com \
  -n fhir-healthcare
```

---

## Incident Response

### Severity Levels

| Level | Description | Response Time | Escalation |
|-------|-------------|---------------|------------|
| P1 - Critical | Service down, data loss | 15 minutes | Immediate |
| P2 - High | Degraded performance | 1 hour | 2 hours |
| P3 - Medium | Minor issues, workaround available | 4 hours | 8 hours |
| P4 - Low | Cosmetic issues, feature requests | 1 business day | N/A |

### Incident Response Procedure

#### 1. Detection & Triage

```bash
# Quick health check
kubectl get pods -n fhir-healthcare
kubectl top pods -n fhir-healthcare
kubectl get events -n fhir-healthcare --sort-by='.lastTimestamp' | tail -20
```

#### 2. Containment

```bash
# If FHIR server is causing issues, scale down
kubectl scale deployment test-fhir-instance-fhir --replicas=0 -n fhir-healthcare

# If database is corrupted, stop writes
kubectl scale deployment test-fhir-instance-fhir --replicas=0 -n fhir-healthcare
```

#### 3. Investigation

```bash
# Collect logs
kubectl logs deployment/test-fhir-instance-fhir -n fhir-healthcare --previous > fhir-crash.log
kubectl logs statefulset/test-fhir-instance-postgres -n fhir-healthcare > postgres.log

# Collect events
kubectl get events -n fhir-healthcare --sort-by='.lastTimestamp' > events.log

# Collect resource usage
kubectl top pods -n fhir-healthcare > resource-usage.log
```

#### 4. Resolution

```bash
# Apply fix (example: rollback)
kubectl rollout undo deployment/test-fhir-instance-fhir -n fhir-healthcare

# Verify resolution
./health-check.sh
```

#### 5. Post-Incident Review

- Document root cause
- Update runbook with lessons learned
- Implement preventive measures
- Update monitoring/alerts

---

## Contact Information

### Support Escalation

| Level | Contact | Response Time |
|-------|---------|---------------|
| L1 - Operations | ops-team@sovereign-core.ibm.com | 15 minutes |
| L2 - Engineering | engineering@sovereign-core.ibm.com | 1 hour |
| L3 - Architecture | architects@sovereign-core.ibm.com | 4 hours |

### On-Call Rotation

- **Primary**: ops-oncall@sovereign-core.ibm.com
- **Secondary**: engineering-oncall@sovereign-core.ibm.com
- **Escalation**: incident-commander@sovereign-core.ibm.com

---

**Runbook Version**: 1.0  
**Last Updated**: 2026-04-26  
**Next Review**: 2026-07-26
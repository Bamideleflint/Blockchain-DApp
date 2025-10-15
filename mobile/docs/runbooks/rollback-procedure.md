# Rollback Procedure

## Table of Contents

- [Overview](#overview)
- [When to Rollback](#when-to-rollback)
- [Pre-Rollback Checklist](#pre-rollback-checklist)
- [Rollback Procedures](#rollback-procedures)
  - [Backend Application Rollback](#backend-application-rollback)
  - [Frontend Application Rollback](#frontend-application-rollback)
  - [Database Migration Rollback](#database-migration-rollback)
  - [Infrastructure Rollback](#infrastructure-rollback)
- [Post-Rollback Verification](#post-rollback-verification)
- [Communication Template](#communication-template)
- [Troubleshooting](#troubleshooting)

## Overview

This runbook provides step-by-step procedures for rolling back deployments in the Blockchain DApp Platform. Rollbacks are critical safety mechanisms when a deployment causes issues in production.

### Rollback Types

| Type | Scope | Complexity | Time Estimate |
|------|-------|------------|---------------|
| **Backend Application** | Kubernetes deployment | Low | 2-5 minutes |
| **Frontend Application** | S3/CloudFront | Low | 3-5 minutes |
| **Database Migration** | Database schema | Medium | 10-30 minutes |
| **Infrastructure** | Terraform resources | High | 15-60 minutes |

### Key Principles

1. **Safety First**: Always verify rollback plan before executing
2. **Communication**: Notify team before and after rollback
3. **Documentation**: Record all actions taken
4. **Verification**: Test thoroughly after rollback
5. **Root Cause**: Investigate why rollback was necessary

## When to Rollback

### Rollback Immediately If:

✅ **Critical Bugs**
- Application crashes or won't start
- Data corruption or loss
- Security vulnerability introduced
- Complete feature failure

✅ **Performance Degradation**
- Response time increased >50%
- Error rate >5%
- Memory leaks causing OOM crashes
- Database deadlocks

✅ **Failed Health Checks**
- Readiness probes failing
- Liveness probes failing
- External health monitoring failing

### Consider Alternatives If:

⚠️ **Minor Issues**
- Cosmetic UI bugs
- Non-critical feature issues
- Issues affecting <1% of users

**Alternatives**:
- Hot-fix deployment
- Feature flag disable
- Configuration change
- Emergency patch

## Pre-Rollback Checklist

Before initiating a rollback, verify:

- [ ] **Incident Severity**: SEV 2 or higher
- [ ] **Root Cause**: Deployment confirmed as cause
- [ ] **Rollback Target**: Previous version identified
- [ ] **Stakeholders Notified**: Team aware of rollback
- [ ] **Backup Verified**: Database backup exists (if applicable)
- [ ] **Rollback Plan**: Procedure reviewed
- [ ] **Communication Ready**: Status page update prepared

## Rollback Procedures

### Backend Application Rollback

Rolling back the Go backend API deployed to Kubernetes (EKS).

#### Quick Rollback (Recommended)

**When to use**: When you need to quickly revert to the previous deployment.

```bash
# 1. Connect to EKS cluster
aws eks update-kubeconfig --name <CLUSTER_NAME> --region us-east-1

# 2. Verify current deployment
kubectl get deployment blockchain-backend -o wide

# Expected output shows current image tag
# NAME                  READY   IMAGE
# blockchain-backend    3/3     <ECR_URL>/blockchain-backend:abc123

# 3. Check rollout history
kubectl rollout history deployment/blockchain-backend

# Expected output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         Deployed version abc123
# 3         Deployed version def456  <- Current (problematic)

# 4. Rollback to previous revision
kubectl rollout undo deployment/blockchain-backend

# 5. Monitor rollback progress
kubectl rollout status deployment/blockchain-backend

# Expected output:
# Waiting for deployment "blockchain-backend" rollout to finish: 1 out of 3 new replicas have been updated...
# deployment "blockchain-backend" successfully rolled out

# 6. Verify pods are running
kubectl get pods -l app=blockchain-backend

# Expected: All pods in Running state
```

**Estimated Time**: 2-3 minutes

#### Rollback to Specific Version

**When to use**: When you need to rollback to a specific version (not just previous).

```bash
# 1. List available revisions
kubectl rollout history deployment/blockchain-backend

# 2. Check specific revision details
kubectl rollout history deployment/blockchain-backend --revision=2

# 3. Rollback to specific revision
kubectl rollout undo deployment/blockchain-backend --to-revision=2

# 4. Monitor rollback
kubectl rollout status deployment/blockchain-backend
```

#### Manual Rollback (Using Image Tag)

**When to use**: When rollout history is unavailable or you know the exact image tag.

```bash
# 1. Identify target image tag (from ECR or git history)
# Example: abc123 (git commit SHA)

# 2. Update deployment image
kubectl set image deployment/blockchain-backend \
  blockchain-backend=<ECR_REGISTRY>/blockchain-backend:abc123

# Alternative: Edit deployment directly
kubectl edit deployment blockchain-backend
# Change spec.template.spec.containers[0].image to target version

# 3. Monitor rollout
kubectl rollout status deployment/blockchain-backend
```

#### Verify Backend Rollback

```bash
# 1. Check application health
curl https://api.yourdomain.com/health

# Expected: {"status": "healthy"}

# 2. Check pod logs for errors
kubectl logs -l app=blockchain-backend --tail=50

# 3. Check version endpoint (if available)
curl https://api.yourdomain.com/version

# 4. Monitor metrics
# - Error rate should be <1%
# - Response time should be normal
# - No crash loops
```

### Frontend Application Rollback

Rolling back the React web application deployed to S3/CloudFront.

#### Option 1: Re-deploy Previous Version

**When to use**: Standard rollback scenario.

```bash
# 1. Navigate to app directory
cd app

# 2. Checkout previous version
git log --oneline  # Find previous good commit
git checkout <COMMIT_HASH>

# Example:
git checkout abc123

# 3. Install dependencies (use exact previous versions)
npm ci

# 4. Build application
npm run build

# 5. Deploy to S3
aws s3 sync dist/ s3://<BUCKET_NAME>/ --delete

# 6. Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"

# 7. Monitor invalidation status
aws cloudfront get-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --id <INVALIDATION_ID>

# 8. Return to main branch
git checkout main
```

**Estimated Time**: 3-5 minutes (plus CloudFront propagation)

#### Option 2: Restore from S3 Versioning

**When to use**: If S3 versioning is enabled (recommended).

```bash
# 1. List object versions
aws s3api list-object-versions \
  --bucket <BUCKET_NAME> \
  --prefix index.html

# 2. Identify previous version ID
# Look for VersionId of previous deployment

# 3. Copy previous version to current
aws s3api copy-object \
  --bucket <BUCKET_NAME> \
  --copy-source <BUCKET_NAME>/index.html?versionId=<VERSION_ID> \
  --key index.html

# 4. Repeat for all changed files (or use script)

# 5. Invalidate CloudFront
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"
```

#### Verify Frontend Rollback

```bash
# 1. Test CloudFront URL
curl https://yourdomain.com

# 2. Check browser
# Open in private/incognito window to bypass cache
# Verify application loads correctly

# 3. Check CloudFront distribution
aws cloudfront get-distribution --id <DISTRIBUTION_ID> | grep Status

# Expected: "Status": "Deployed"

# 4. Check browser console for errors
# Should be no JavaScript errors
```

### Database Migration Rollback

Rolling back database schema changes (most complex rollback).

⚠️ **WARNING**: Database rollbacks can cause data loss. Always take a snapshot first.

#### Pre-Rollback: Create Snapshot

```bash
# 1. Create manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-postgres \
  --db-snapshot-identifier rollback-$(date +%Y%m%d-%H%M%S)

# 2. Wait for snapshot to complete
aws rds describe-db-snapshots \
  --db-snapshot-identifier rollback-20240115-143000 \
  --query 'DBSnapshots[0].Status'

# Expected: "available"
```

#### Option 1: Run Down Migration (Preferred)

**When to use**: When down migrations exist and are tested.

```bash
# Example using golang-migrate

# 1. Connect to database (use bastion/jump host)
ssh bastion-host

# 2. Check current migration version
migrate -path db/migrations -database "postgres://..." version

# Expected: Current version number (e.g., 20240115143000)

# 3. Rollback one migration
migrate -path db/migrations -database "postgres://..." down 1

# 4. Verify migration
migrate -path db/migrations -database "postgres://..." version

# 5. Test database queries
psql -h <RDS_ENDPOINT> -U postgres -d blockchain_dapp
# Run test queries to verify schema
```

#### Option 2: Restore from Snapshot (Last Resort)

**When to use**: When down migration failed or doesn't exist.

⚠️ **WARNING**: This will lose all data written since snapshot was taken.

```bash
# 1. Identify snapshot to restore
aws rds describe-db-snapshots \
  --db-instance-identifier prod-postgres \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime]' \
  --output table

# 2. Restore snapshot to new instance
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-postgres-restored \
  --db-snapshot-identifier <SNAPSHOT_ID>

# 3. Wait for instance to be available (10-20 minutes)
aws rds wait db-instance-available \
  --db-instance-identifier prod-postgres-restored

# 4. Verify data in restored instance
psql -h <NEW_ENDPOINT> -U postgres -d blockchain_dapp
# Run verification queries

# 5. Update application connection string
kubectl edit secret db-credentials
# Update DB_HOST to new endpoint

# 6. Restart application
kubectl rollout restart deployment/blockchain-backend

# 7. Verify application works
curl https://api.yourdomain.com/health

# 8. After verification, delete old instance
aws rds delete-db-instance \
  --db-instance-identifier prod-postgres \
  --skip-final-snapshot

# 9. Rename restored instance (optional)
aws rds modify-db-instance \
  --db-instance-identifier prod-postgres-restored \
  --new-db-instance-identifier prod-postgres \
  --apply-immediately
```

**Estimated Time**: 15-30 minutes (migration) or 30-60 minutes (snapshot restore)

### Infrastructure Rollback

Rolling back Terraform infrastructure changes.

⚠️ **WARNING**: Infrastructure rollbacks can be destructive. Review plan carefully.

#### Git-Based Rollback

```bash
# 1. Navigate to infrastructure directory
cd infra/env/prod

# 2. Check terraform state
terraform show

# 3. Identify problematic commit
git log --oneline infra/

# 4. Checkout previous version
git checkout <COMMIT_HASH> -- infra/

# Example:
git checkout abc123 -- infra/

# 5. Review plan (CRITICAL)
terraform plan

# Carefully review:
# - Resources to be destroyed (should be minimal)
# - Resources to be modified
# - Potential data loss

# 6. If plan looks safe, apply
terraform apply

# 7. Verify infrastructure
kubectl get nodes  # For EKS changes
aws rds describe-db-instances  # For RDS changes

# 8. Return to main branch
cd ../../..
git checkout main
```

#### State-Based Rollback (Advanced)

**When to use**: When you need to rollback specific resource without git.

```bash
# 1. List terraform state backups
ls -la terraform.tfstate.backup*

# 2. Review backup state
terraform show terraform.tfstate.backup

# 3. Restore backup (DANGEROUS)
cp terraform.tfstate terraform.tfstate.broken
cp terraform.tfstate.backup terraform.tfstate

# 4. Verify state
terraform plan

# 5. If needed, apply
terraform apply
```

⚠️ **Note**: Only use state rollback as last resort. Prefer git-based rollback.

## Post-Rollback Verification

### Comprehensive Health Check

```bash
#!/bin/bash
# save as health-check.sh

echo "=== 1. API Health ==="
curl -f https://api.yourdomain.com/health || echo "FAILED"

echo "\n=== 2. Frontend Health ==="
curl -f https://yourdomain.com || echo "FAILED"

echo "\n=== 3. Backend Pods ==="
kubectl get pods -l app=blockchain-backend | grep Running || echo "FAILED"

echo "\n=== 4. Recent Backend Logs ==="
kubectl logs -l app=blockchain-backend --tail=20 | grep -i error || echo "No errors"

echo "\n=== 5. Database Connectivity ==="
kubectl exec -it deployment/blockchain-backend -- psql -h <DB_HOST> -U postgres -c "SELECT 1" || echo "FAILED"

echo "\n=== 6. Redis Connectivity ==="
kubectl exec -it deployment/blockchain-backend -- redis-cli -h <REDIS_HOST> PING || echo "FAILED"

echo "\n=== 7. Application Metrics ==="
echo "Check dashboards for:"
echo "  - Error rate < 1%"
echo "  - Response time < 200ms p95"
echo "  - No alerts firing"
```

### Verification Checklist

- [ ] **Application Health**: Health endpoints return 200 OK
- [ ] **Pods Running**: All pods in Running state, no CrashLoopBackOff
- [ ] **No Errors**: Logs show no critical errors
- [ ] **Database Connected**: Application can query database
- [ ] **Cache Connected**: Application can access Redis
- [ ] **User Flow**: Critical user journeys work (login, transaction, etc.)
- [ ] **Metrics Normal**: Error rate, latency, throughput within normal range
- [ ] **Alerts Clear**: No active alerts

### Monitoring Period

**Monitor for at least 30 minutes after rollback**:

```bash
# Monitor pods
watch kubectl get pods -l app=blockchain-backend

# Monitor logs
kubectl logs -f deployment/blockchain-backend

# Monitor metrics (if Prometheus/Grafana available)
# Check dashboards for sustained stability
```

## Communication Template

### Pre-Rollback Announcement

**Slack/Team Channel**:
```
[ACTION] Initiating Rollback - Backend Deployment

Reason: Elevated error rate (15%) after deployment at 14:30 UTC
Target: Rollback to version abc123 (previous stable)
Impact: Brief service disruption (2-3 minutes)
Executor: @john.doe
Time: Starting immediately

Status page updated: https://status.yourdomain.com
```

### Post-Rollback Announcement

**Slack/Team Channel**:
```
[COMPLETE] Rollback Successful

Rollback completed: 14:45 UTC
Duration: 3 minutes
Current version: abc123 (stable)
Status: All systems operational

Verification:
✅ API health: OK
✅ Error rate: <1%
✅ Pods: All running
✅ Database: Connected

Monitoring for 30 minutes. Post-incident review scheduled.
```

**Status Page Update**:
```
Title: Service Restored

We have rolled back a recent deployment that caused elevated error rates. 
All services are now operating normally.

Updated: 2024-01-15 14:45 UTC
```

## Troubleshooting

### Rollback Won't Complete

**Symptom**: Pods stuck in pending or terminating

**Solution**:
```bash
# Check pod status
kubectl describe pod <POD_NAME>

# Force delete stuck pods
kubectl delete pod <POD_NAME> --grace-period=0 --force

# Check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"

# If nodes full, scale up
# Via AWS Console or eksctl
```

### Rollback Causes Different Errors

**Symptom**: Rolled back, but seeing new errors

**Possible Causes**:
- Database schema incompatible with old code
- Environment variables changed
- External dependency changed

**Solution**:
```bash
# Check logs for specific error
kubectl logs -l app=blockchain-backend | grep ERROR

# Compare ConfigMaps
kubectl get configmap app-config -o yaml

# Compare Secrets
kubectl get secret db-credentials -o yaml

# May need to roll forward with hot-fix instead
```

### Can't Find Previous Version

**Symptom**: Rollout history empty or ECR image deleted

**Solution**:
```bash
# Check ECR for available images
aws ecr describe-images \
  --repository-name blockchain-backend \
  --query 'imageDetails[*].[imageTags[0],imagePushedAt]' \
  --output table

# Check git history for commit SHAs
git log --oneline -20

# Use git SHA to find corresponding image
# Images tagged with git commit hash
```

### Database Rollback Failed

**Symptom**: Migration down failed, database in inconsistent state

**Solution**:
```bash
# 1. Don't panic - snapshot exists

# 2. Assess damage
psql -h <RDS_ENDPOINT> -U postgres -d blockchain_dapp
# Check tables, run queries

# 3. If fixable, apply manual SQL
# (with extreme caution)

# 4. If not fixable, restore from snapshot
# See "Restore from Snapshot" procedure above

# 5. Document incident for post-mortem
```

## Prevention Best Practices

### Reduce Need for Rollbacks

1. **Test Thoroughly**
   - Unit tests
   - Integration tests
   - Load tests
   - Staging deployment

2. **Deploy Incrementally**
   - Deploy to dev first
   - Then staging
   - Finally production

3. **Use Feature Flags**
   - Enable new features gradually
   - Instant disable without deployment

4. **Canary Deployments** (Future)
   - Route small percentage of traffic to new version
   - Monitor metrics
   - Rollout or rollback based on metrics

5. **Database Migrations**
   - Always write down migrations
   - Test down migrations in dev
   - Make schema changes backward compatible
   - Deploy schema changes separately from code changes

### Keep Rollback Options Available

1. **Retain Old Images**
   ```bash
   # Set ECR lifecycle policy to keep last 30 images
   # Don't delete old images immediately
   ```

2. **Tag Everything**
   ```bash
   # Always tag with git SHA
   docker build -t app:$(git rev-parse --short HEAD)
   
   # Keep semantic versions
   docker tag app:abc123 app:v1.2.3
   ```

3. **Maintain Rollout History**
   ```yaml
   # Set revisionHistoryLimit in deployment
   spec:
     revisionHistoryLimit: 10
   ```

4. **Enable S3 Versioning**
   ```bash
   aws s3api put-bucket-versioning \
     --bucket <BUCKET_NAME> \
     --versioning-configuration Status=Enabled
   ```

## Related Documentation

- [Incident Response](incident-response.md)
- [Scaling Guide](scaling-guide.md)
- [System Overview](../architecture/system-overview.md)
- [Developer Setup](../onboarding/dev-setup.md)

---

**Remember**: A rollback is a tool, not a failure. Better to rollback quickly and investigate safely than to debug in production.
# Swiggy Scale Incident Runbook 📋

**Version:** 1.0  
**Last Updated:** May 2024  
**Audience:** On-call engineers (first-year engineers can follow this)  
**Severity:** Critical incidents (5xx errors > 5% for 2+ minutes)

---

## STEP 1 - DETECT: CloudWatch Alarm Thresholds

These alarms must be configured BEFORE an incident happens. Each alarm has a metric, threshold, and alert level.

### Critical Alarms (Page immediately)

| # | Alarm Name | Metric | Threshold | Duration | Alert Type | Action |
|---|-----------|--------|-----------|----------|-----------|--------|
| 1 | **ALB 5xx Error Rate High** | `TargetResponseTime` metric or error count | > 5% 5xx for 2 min | 2 minutes | 🔴 CRITICAL | Page on-call immediately |
| 2 | **RDS Connections High** | `DatabaseConnections` | > 80 of 100 max | 1 minute | 🔴 CRITICAL | DB pool near exhaustion |
| 3 | **RDS CPU High** | `CPUUtilization` | > 80% | 2 minutes | 🟠 WARNING | Primary under load |
| 4 | **Node.js App CPU** | `CPUUtilization` (all instances) | > 80% for all | 2 minutes | 🔴 CRITICAL | Compute saturation starting |
| 5 | **Node.js Memory** | `MemoryUtilization` | > 85% on any instance | 1 minute | 🟠 WARNING | OOM risk in 5-10 min |
| 6 | **SQS Queue Depth** | `ApproximateNumberOfMessages` | > 10,000 | 2 minutes | 🟠 WARNING | Payment backlog forming |
| 7 | **Redis Cache Misses** | `CacheMisses` / `CacheHits` ratio | > 50% miss rate | 3 minutes | 🟠 WARNING | Cache invalidated or eviction |
| 8 | **ALB Target Unhealthy** | `UnHealthyHostCount` | > 0 on any target group | 30 seconds | 🟠 WARNING | Instance health check failed |
| 9 | **Response Time P99** | `TargetResponseTime` p99 | > 2000ms | 2 minutes | 🟠 WARNING | User experience degrading |
| 10 | **Promo Budget Remaining** | Custom metric `PromoPercentRemaining` | < 10% | 1 minute | 🟠 WARNING | Pause promo distribution |

### Setup Commands (Run Once)

```bash
# Create alarm for ALB 5xx errors
aws cloudwatch put-metric-alarm \
  --alarm-name swiggy-alb-5xx-high \
  --alarm-description "ALB 5xx error rate > 5% for 2 minutes" \
  --metric-name HTTPCode_Target_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 60 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:swiggy-critical

# Create alarm for DB connections high
aws cloudwatch put-metric-alarm \
  --alarm-name swiggy-rds-connections-high \
  --alarm-description "RDS connections > 80" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --statistic Average \
  --period 60 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:swiggy-warning
```

---

## STEP 2 - TRIAGE: Root Cause Decision Tree (30 Seconds)

**You have been paged. Follow this decision tree in order. The FIRST alarm that shows red is your root cause.**

```
START HERE ↓

┌─────────────────────────────────────────────────────────┐
│ CHECK 1: ALB Target Response Time & 5xx Rate           │
│ Command: aws cloudwatch get-metric-statistics \         │
│  --metric-name HTTPCode_Target_5XX_Count \              │
│  --start-time $(date -u -d '5 min ago' +%Y-%m-%dT%H:%M:%S) \
│  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
│  --period 60 --statistics Sum \                         │
│  --namespace AWS/ApplicationELB                         │
│                                                          │
│ Result:                                                 │
│  RED (5xx > 50/min)? → Go to CHECK 2 (DB or Compute)  │
│  YELLOW (p99 > 2s)?  → Go to CHECK 3 (Load)           │
│  GREEN?              → Go to CHECK 5 (Cache/Queue)     │
└─────────────────────────────────────────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        ▼                                  ▼
   CHECK 2                            CHECK 3
   
┌──────────────────────────────┐  ┌──────────────────────────────┐
│ Is it DB Pool Exhaustion?    │  │ Is it Compute Saturation?    │
│                              │  │                              │
│ Command:                     │  │ Command:                     │
│ psql -h <rds-primary> \      │  │ aws ec2 describe-instances \ │
│  -U postgres -d swiggy \     │  │  --query 'Reservations[*]... │
│  -c "SELECT count(*) FROM   │  │   Instances[?Tags[?Key==     │
│   pg_stat_activity;"         │  │   `swiggy-app`]].          │
│                              │  │   [CPUUtilization]'        │
│ Result:                      │  │                            │
│ > 90 connections?            │  │ Result:                    │
│   → FAILURE 3A: DB POOL      │  │ All instances > 80% CPU?  │
│                              │  │   → FAILURE 3B: COMPUTE   │
│ 50-89 connections?           │  │                            │
│   → Saturating soon          │  │ Some instances > 80%?      │
│   → Go to CHECK 4            │  │   → FAILURE 3B variant     │
│                              │  │                            │
│ < 50?                        │  │ All instances < 70%?       │
│   → Go to CHECK 4            │  │   → Go to CHECK 4          │
└──────────────────────────────┘  └──────────────────────────────┘
        │                                    │
        └────────────────┬───────────────────┘
                         │
                    CHECK 4
                    
┌──────────────────────────────────────────────────────────┐
│ Is it a Cache Issue?                                    │
│                                                          │
│ Command:                                                │
│ redis-cli -h <redis-endpoint> INFO stats | grep \       │
│  "evicted_keys"                                         │
│                                                          │
│ Result:                                                 │
│ evicted_keys > 1000?                                    │
│   → FAILURE 3C: CACHE MISS / EVICTION                   │
│                                                          │
│ evicted_keys < 1000?                                    │
│   → Go to CHECK 5                                       │
└──────────────────────────────────────────────────────────┘
                         │
                    CHECK 5
                    
┌──────────────────────────────────────────────────────────┐
│ Is it a Payment Queue Issue?                             │
│                                                          │
│ Command:                                                │
│ aws sqs get-queue-attributes \                          │
│  --queue-url <SQS-QUEUE-URL> \                          │
│  --attribute-names ApproximateNumberOfMessages          │
│                                                          │
│ Result:                                                 │
│ > 10,000 messages?                                      │
│   → FAILURE 3D: PAYMENT QUEUE BACKUP                    │
│                                                          │
│ < 10,000 messages?                                      │
│   → Unknown issue → Escalate to @devops-lead            │
└──────────────────────────────────────────────────────────┘

END TRIAGE: You have identified the root cause.
Go to Step 3 with the failure type.
```

### Triage Output Template

**Copy this to Slack #incident-channel when you identify the cause:**

```
🔍 TRIAGE COMPLETE (T+[time])
Root Cause: [Failure 3A / 3B / 3C / 3D / Unknown]
Affected Component: [RDS / EC2 / Redis / SQS]
Severity: [CRITICAL / WARNING]
Next Step: Proceeding to RESPOND → [Step 3a/3b/3c/3d]
```

---

## STEP 3 - RESPOND: Component-Specific Action Steps

### FAILURE 3A: PostgreSQL Connection Pool Exhausted

**Symptoms:** 
- RDS connections > 90
- All API requests returning "connection timeout"
- New connections rejected

**Owner:** @dba-oncall  
**Slack Channel:** #database-incidents

#### Immediate Actions (T+0s to T+2min)

```bash
# STEP 1: Connect to primary database and inspect
psql -h swiggy-primary.us-east-1.rds.amazonaws.com \
     -U postgres -d swiggy

# STEP 2: Check connection states (run inside psql)
SELECT 
  state, 
  usename,
  query_start,
  COUNT(*) as connection_count
FROM pg_stat_activity
GROUP BY state, usename, query_start
ORDER BY connection_count DESC;

# STEP 3: Identify stale connections (idle for > 30s)
SELECT 
  pid,
  usename,
  state,
  query_start,
  state_change
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND query_start < NOW() - INTERVAL '30 seconds'
LIMIT 20;

# STEP 4: Terminate stale connections
-- WARNING: This will cancel active transactions!
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND query_start < NOW() - INTERVAL '30 seconds';

# After terminating, verify connection count dropped
SELECT count(*) as total_connections FROM pg_stat_activity;
```

#### If connections still high (T+2min to T+5min)

```bash
# STEP 5: Increase PgBouncer pool size (temporary)
ssh pgbouncer-instance
sudo nano /etc/pgbouncer/pgbouncer.ini

# Change:
#   pool_size = 100
# To:
#   pool_size = 150

# Restart PgBouncer
sudo systemctl restart pgbouncer

# Verify it started
sudo systemctl status pgbouncer
```

#### If still saturated (T+5min to T+10min)

```bash
# STEP 6: Failover to standby (LAST RESORT)
# WARNING: All connections will briefly drop
# Only do this if primary is permanently stuck

aws rds failover-db-cluster \
  --db-cluster-identifier swiggy-cluster \
  --region us-east-1

# Monitor promotion
aws rds describe-db-instances \
  --db-instance-identifier swiggy-primary \
  --query 'DBInstances[0].DBInstanceStatus'

# New primary endpoint appears in AWS Console
# Update Parameter Store (will auto-refresh app connections)
aws ssm put-parameter \
  --name /swiggy/rds/primary-endpoint \
  --value <new-primary-endpoint> \
  --overwrite
```

**Success Criteria:**
- RDS connections drop to < 50 within 2 minutes
- API 5xx error rate drops to < 1%
- P99 response time returns to < 500ms

**Who to notify:**
- Slack: @dba-oncall
- Email: dba-team@swiggy.internal
- Page: PagerDuty escalation policy #database

---

### FAILURE 3B: Node.js Compute Saturation

**Symptoms:**
- All EC2 instances showing CPU > 80%
- ALB health checks failing (instances become unavailable)
- Requests timing out

**Owner:** @app-oncall  
**Slack Channel:** #application-incidents

#### Immediate Actions (T+0s to T+2min)

```bash
# STEP 1: Verify all instances are overloaded
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=swiggy-app" \
  --query 'Reservations[*].Instances[*].[InstanceId,LaunchTime,Placement.AvailabilityZone,State.Name]' \
  --output table

# Get CPU utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=swiggy-app-asg \
  --start-time $(date -u -d '5 min ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average,Maximum

# STEP 2: Trigger manual auto-scaling (scale up NOW)
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name swiggy-app-asg \
  --desired-capacity 20 \
  --region us-east-1

# STEP 3: Monitor new instances launching
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names swiggy-app-asg \
  --query 'AutoScalingGroups[0].[DesiredCapacity,Instances[].{InstanceId:InstanceId,HealthStatus:HealthStatus}]'

# Watch for instances becoming InService (takes ~90 seconds)
watch -n 5 'aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names swiggy-app-asg \
  --query "AutoScalingGroups[0].Instances[].[InstanceId,HealthStatus]" \
  --output table'
```

#### Monitor Recovery (T+2min to T+5min)

```bash
# STEP 4: Check ALB target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:ACCOUNT:targetgroup/swiggy-app/xyz \
  --query 'TargetHealthDescriptions[?TargetHealth.State==`healthy`]' \
  | wc -l

# Expected: Number should increase from ~4 to ~10 then to ~20

# STEP 5: Verify 5xx error rate dropping
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --start-time $(date -u -d '2 min ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum

# Should see: 50 → 30 → 10 → 0 over next 3 minutes
```

**Success Criteria:**
- Desired capacity increases from 4 to 20 instances
- Healthy target count reaches 20 within 90 seconds
- CPU drops to 40-60% across all instances
- 5xx error rate drops to < 1% within 3 minutes

**Who to notify:**
- Slack: @app-oncall
- Page: PagerDuty application escalation
- If scaling doesn't help: Escalate to @infrastructure-team

---

### FAILURE 3C: Redis Cache Miss Rate High / Memory Eviction

**Symptoms:**
- Cache hit rate drops from 80% to < 50%
- Redis memory > 95%
- evicted_keys climbing

**Owner:** @cache-oncall  
**Slack Channel:** #infrastructure

#### Immediate Actions (T+0s to T+3min)

```bash
# STEP 1: Connect to Redis and inspect
redis-cli -h swiggy-cache.us-east-1.cache.amazonaws.com

# Inside redis-cli, check memory
INFO memory

# Look for:
# - used_memory_human
# - maxmemory
# - evicted_keys

# STEP 2: Check eviction policy
CONFIG GET maxmemory-policy

# Should be: allkeys-lru (evict least recently used)

# STEP 3: Check key count
DBSIZE

# STEP 4: Diagnose cause
# A) Was there a recent deploy with FLUSHALL?
#    → Wait 10-15 minutes for cache to rebuild
#    → Go to STEP 5
#
# B) Is memory genuinely full (normal spike)?
#    → Proceed to STEP 6 (scale up cluster)

# STEP 5: If due to FLUSHALL (cache rebuilding)
# Watch cache hit rate recovery
MONITOR  # (Ctrl+C to exit)

# Should see hit rate improving over 10-15 minutes
# Estimated recovery time: 1 request per menu item × 50K items
```

#### Scale Up Redis (T+3min to T+10min)

```bash
# STEP 6: Add a shard to ElastiCache cluster
# NOTE: This requires AWS Console (CLI doesn't support add-shard for now)

# Via AWS CLI: Create new node in cluster
aws elasticache modify-replication-group \
  --replication-group-id swiggy-cache \
  --num-cache-clusters 4 \
  --apply-immediately \
  --region us-east-1

# Monitor scaling progress
watch -n 10 'aws elasticache describe-replication-groups \
  --replication-group-id swiggy-cache \
  --query "ReplicationGroups[0].[Status,MemberClusters]"'

# Scaling takes 10-20 minutes, zero downtime
# Cluster will rebalance keys across new node
```

#### Fallback: Manual Cache Clear (ONLY if needed)

```bash
# ⚠️ WARNING: This will clear all cached data!
# Only do if cache is corrupt or growth is uncontrolled

redis-cli -h swiggy-cache.us-east-1.cache.amazonaws.com FLUSHALL

# This will trigger immediate cache miss spike (expected)
# Cache will rebuild from database in 10-15 minutes
# API will see 200-500ms latency during rebuild
```

**Success Criteria:**
- Cache memory drops to < 80% of maxmemory
- evicted_keys stops climbing
- Cache hit rate returns to > 70%
- P99 response time increases to 300-500ms (temporary, rebuilding)

**Who to notify:**
- Slack: @cache-oncall
- Page: PagerDuty infrastructure escalation

---

### FAILURE 3D: Payment Queue Backed Up (SQS Depth > 10K)

**Symptoms:**
- SQS ApproximateNumberOfMessages > 10,000
- Payment processing delayed
- Users seeing "payment processing" status for > 30 seconds

**Owner:** @payments-oncall  
**Slack Channel:** #payment-incidents

#### Immediate Actions (T+0s to T+2min)

```bash
# STEP 1: Check queue depth and DLQ
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT/swiggy-payment-queue \
  --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfDelayedMessages

# Also check DLQ
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT/swiggy-payment-dlq \
  --attribute-names ApproximateNumberOfMessages

# STEP 2: Diagnose why queue is growing
# Option A: Payment worker crashed
aws ecs describe-services \
  --cluster swiggy-workers \
  --services payment-worker \
  --query 'services[0].{Status:Status,DesiredCount:DesiredCount,RunningCount:RunningCount}'

# If RunningCount < DesiredCount:
#   → Worker crashed, proceed to STEP 3

# Option B: Payment gateway is rate-limited or down
# Check CloudWatch logs
aws logs tail /ecs/payment-worker --follow --since 5m

# Look for:
# - "Razorpay 429" (rate limited)
# - "Razorpay 5xx" (gateway down)
# - "Connection timeout" (network issue)

# STEP 3: Restart payment worker (if crashed)
aws ecs update-service \
  --cluster swiggy-workers \
  --service payment-worker \
  --force-new-deployment \
  --region us-east-1

# Monitor restart (takes 30-60 seconds)
watch -n 5 'aws ecs describe-services \
  --cluster swiggy-workers \
  --services payment-worker \
  --query "services[0].{DesiredCount:DesiredCount,RunningCount:RunningCount}"'

# STEP 4: Check if worker is processing messages
# Query payment_logs table to see if processing rate > 0
psql -h swiggy-primary.rds.amazonaws.com \
     -U postgres -d swiggy \
     -c "SELECT 
           DATE_TRUNC('minute', processed_at) as minute,
           COUNT(*) as processed_count
         FROM payment_transactions
         WHERE processed_at > NOW() - INTERVAL '10 minutes'
         GROUP BY minute
         ORDER BY minute DESC
         LIMIT 10;"

# Should see recent rows with processed_count > 0
# If all zeros: Worker is still not processing, escalate
```

#### If Queue Still Growing (T+2min to T+5min)

```bash
# STEP 5: Check if it's a gateway rate limit issue
# Contact Razorpay support team
# Slack: @razorpay-api-support

# Temporary: Increase payment worker concurrency
aws ecs update-service \
  --cluster swiggy-workers \
  --service payment-worker \
  --task-definition payment-worker:LATEST \
  --region us-east-1

# Update the task definition to increase:
# CONCURRENCY_LIMIT = 10 (from 5)
# In ECS console: Task definition → payment-worker → Create new revision
```

#### If Razorpay is Down (T+5min)

```bash
# STEP 6: Pause order placement temporarily
# Update ALB to return 503 for POST /orders

# Option A: Quick fix - modify app code
# SET feature flag: ORDERS_PAUSED = true
aws ssm put-parameter \
  --name /swiggy/feature/orders-paused \
  --value "true" \
  --overwrite

# Apps will check this and show: "Order temporarily unavailable"

# Option B: Block at ALB level (if app is also down)
# Modify ALB listener rules to reject POST /orders

# Notify users via in-app banner
# POST /notifications/broadcast with message:
#   "Payment gateway temporarily down. Please try again in 5 minutes."
```

**Success Criteria:**
- SQS depth drops from 10K+ to < 1K within 5 minutes
- DLQ remains empty (no poison messages)
- Payment worker restart shows RunningCount == DesiredCount
- Payment processing rate > 100 transactions/minute

**Who to notify:**
- Slack: @payments-oncall
- Page: PagerDuty payments escalation
- If gateway issue: @razorpay-api-support

---

## STEP 4 - ROLLBACK: When and How

### Rollback Decision Matrix

**Roll back when ALL of these are true:**

```
Criteria for Rollback:
1. 5xx error rate > 20% AND not improving after 5 minutes of remediation
2. No active deploy in the last 2 hours (not a new code issue)
3. Root cause is NOT external dependency (not Razorpay down, not AWS region issue)
4. Team lead approves (@infrastructure-lead, @app-lead)
```

**If ANY of these are true, do NOT roll back:**
- Active deploy in last 2 hours (revert deployment, not rollback)
- Root cause is external (wait for dependency to recover)
- Only 1 or 2 instances affected (scale/restart instead)
- Error rate is < 5% and stable (not critical enough)

### Rollback Procedure

```bash
# STEP 1: Get the last known good deployment
aws ecs describe-services \
  --cluster swiggy-prod \
  --services api \
  --query 'services[0].deployments' \
  --output table

# Look for the PREVIOUS completed deployment
# Note its task definition: swiggy-api:N (where N is version)

# STEP 2: Confirm with team lead
# Slack: @app-lead "Rolling back from swiggy-api:150 to swiggy-api:149. Confirmed?"
# Wait for thumbs up emoji ✅

# STEP 3: Execute rollback
aws ecs update-service \
  --cluster swiggy-prod \
  --service api \
  --task-definition swiggy-api:149 \
  --force-new-deployment \
  --region us-east-1

# STEP 4: Monitor rollback progress
watch -n 5 'aws ecs describe-services \
  --cluster swiggy-prod \
  --services api \
  --query "services[0].deployments[*].[TaskDefinition,Status,RunningCount,DesiredCount]" \
  --output table'

# Rollback complete when old task definition shows:
# Status=PRIMARY, RunningCount=DesiredCount

# STEP 5: Verify stability
# Check error rate (should drop immediately)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --start-time $(date -u -d '2 min ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum
```

### ⚠️ CRITICAL WARNINGS

**NEVER rollback the database schema:**
```
BAD:  aws rds restore-db-instance-from-db-snapshot \
      --db-instance-identifier swiggy-restored \
      --db-snapshot-identifier swiggy-snapshot-old

GOOD: Only roll back APPLICATION code (ECS task definition)
      Database migrations are one-way (forward only)
```

**Database-level recovery:**
```
If a database migration caused the issue:
1. Stop the current deploy (don't roll back DB)
2. Roll back app code to previous task definition
3. Fix the migration script
4. Re-deploy app only (DB stays at new schema)
5. Test thoroughly before rolling forward
```

**Rollback Timeline:**
- Decision + approval: 2-3 minutes
- Task definition rollback: 30-60 seconds
- New tasks launching: 60-90 seconds
- Error rate recovery: 1-2 minutes
- **Total MTTD: 5-8 minutes**

---

## STEP 5 - POSTMORTEM: Incident Report Template

**Complete this within 24 hours of incident resolution.**

### Postmortem Document Template

```markdown
# Incident Postmortem: [Incident Title]

## Incident Summary

**Date:** [YYYY-MM-DD]
**Start Time:** [HH:MM UTC]
**End Time:** [HH:MM UTC]
**Duration:** [X hours, Y minutes]
**Severity:** [CRITICAL / HIGH / MEDIUM]
**Impacted Users:** [Estimated percentage or count]
**Revenue Impact:** [₹X crore or N/A]

---

## Timeline

**T+0m: [00:05 UTC]**
- Incident triggered: [What happened]
- Detection method: [Alert fired / Manual observation]
- First responder: [@username]

**T+5m: [00:10 UTC]**
- Triage complete: Root cause identified as [Failure Type]
- Assigned to: [@oncall-team]

**T+10m: [00:15 UTC]**
- First remediation action taken: [Action description]
- Expected impact: [What should happen next]

**T+15m: [00:20 UTC]**
- [Continue with specific actions and outcomes]

**T+45m: [00:50 UTC]**
- Incident resolved: [Error rate returned to < 1%, users unaffected]
- Final action: [Verification step]

---

## Root Cause Analysis

**Root Cause:** [Single deepest technical cause, not "server crashed"]

Example (NOT this shallow):
❌ BAD: "Database crashed"
❌ BAD: "High memory usage"
✅ GOOD: "PgBouncer pool_size not updated after RDS instance upgrade, causing connection exhaustion at 394 RPS during World Cup traffic spike (500K RPS incoming)"

**Contributing Factors:**
- [Factor 1: Specific configuration issue]
- [Factor 2: Specific monitoring gap]
- [Factor 3: Specific operational procedure gap]

**Why This Happened:**
- [Explanation of the causal chain]
- [Why wasn't it caught before production]

---

## Impact Assessment

**User Impact:**
- Duration: [X minutes]
- Users affected: [Percentage or count]
- Feature affected: [Order placement / Menu browsing / Payment]
- User-facing behavior: [What users saw: "500 Error", timeouts, etc.]

**Business Impact:**
- Estimated revenue loss: ₹[X crore] or N/A
- Orders affected: [Count]
- App Store reviews: [Impact if any]
- Churn risk: [Low / Medium / High]

**System Impact:**
- Components affected: [Database / Compute / Cache / Queue]
- Data loss: [None / Partial / Complete]
- Data consistency: [All good / Some transactions need reconciliation]

---

## What Went Well ✅

- [@person] quickly identified root cause (30-second triage)
- Alert system worked (Page was fired within 2 minutes)
- Runbook was clear enough to execute quickly
- [List 2-3 things that enabled fast recovery]

---

## What Could Have Gone Better ❌

- Monitoring alert for [metric] was not configured (delayed detection by X minutes)
- [System component] not documented, took 5 minutes to understand
- No [specific procedure] in runbook, wasted time trying [wrong approach]
- [Communication gap: what could have been faster]

---

## Action Items (Next Steps)

| # | Action | Owner | Due Date | Status |
|---|--------|-------|----------|--------|
| 1 | [Create alert for PgBouncer pool_size after RDS upgrade] | @dba-lead | 2024-05-15 | Not Started |
| 2 | [Update runbook with new triage step for connection exhaustion] | @app-lead | 2024-05-15 | Not Started |
| 3 | [Add health check for payment worker in ECS] | @devops-team | 2024-05-17 | Not Started |
| 4 | [Test runbook with first-year engineer (runbook test)] | @sre-lead | 2024-05-16 | Not Started |
| 5 | [Schedule load test for 500K RPS scenario] | @performance-team | 2024-05-20 | Not Started |

---

## Lessons Learned

**For this team:**
- [Key insight about how the system behaves under load]
- [Key insight about monitoring gaps]
- [Key insight about communication during incidents]

**For the broader org:**
- [If this is not specific to this team, what does the org learn]

---

## Sign-off

- **Incident Commander:** [@name] — Approved [Date]
- **Engineering Lead:** [@name] — Approved [Date]
- **On-Call Manager:** [@name] — Approved [Date]
```

### Postmortem Meeting (Schedule within 48 hours)

**Attendees:**
- Incident commander
- All on-call responders
- Engineering leads (app, db, infrastructure)
- Product manager (for impact discussion)

**Duration:** 30-45 minutes

**Agenda:**
1. Walk through timeline (10 min)
2. Root cause discussion (10 min)
3. Action items review (10 min)
4. Commit to action owners (5 min)

**No blame:** Postmortem is blameless. Focus on "what can we learn" not "who messed up".

---

## Quick Reference Cards (Print & Post)

### Card 1: Alert Meanings

```
🔴 CRITICAL
   5xx error rate > 5% for 2 minutes
   ACTION: Page on-call, start STEP 2 triage immediately

🟠 WARNING
   5xx error rate > 2% for 2 minutes
   OR any alarm with 🟠 label
   ACTION: On-call checks, may page if worsening

🟢 NORMAL
   All metrics green
   ACTION: No action needed, carry on
```

### Card 2: Triage Checklist (Laminated)

```
T+0m  [ ] Page received - note timestamp
T+0m  [ ] Check ALB 5xx rate (CHECK 1)
T+1m  [ ] Check RDS connections (CHECK 2)
T+1m  [ ] Check EC2 CPU (CHECK 2)
T+2m  [ ] Check Redis cache hits (CHECK 4)
T+2m  [ ] Check SQS queue depth (CHECK 5)
T+3m  [ ] Identify root cause → Go to Step 3X
```

### Card 3: Emergency Contacts

```
App On-Call:         @app-oncall
Database On-Call:    @dba-oncall
Infrastructure:      @infrastructure-lead
Payments Team:       @payments-oncall
Razorpay Support:    [phone] [email]
AWS Support:         [AWS premium support phone]
```

---

## Final Note for Junior Engineers

**You are reading this because you are on-call. You are ready.**

This runbook exists because the team has already:
- Designed a system that fails predictably
- Documented what each failure looks like
- Provided exact steps to fix each failure
- Monitored all the key signals

**Your job is simple:**
1. Get paged (it will happen)
2. Follow STEP 2 (triage - 30 seconds)
3. Follow STEP 3 (respond - 5 minutes)
4. Watch the system recover
5. Sleep well knowing you fixed a production incident

**If you get stuck:** Slack @app-lead or call the escalation number. You are never alone.

Good luck. You've got this. 💪

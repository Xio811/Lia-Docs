# Maintenance Guide

## Overview

This guide provides comprehensive procedures for maintaining, troubleshooting, and optimizing the LIA chatbot system. It covers routine maintenance tasks, common issues, and best practices for system administrators and DevOps teams.

## Routine Maintenance Tasks

### Daily Tasks

#### 1. Monitor System Health

```bash
# Check application status
kubectl get pods -n lia-production
kubectl get services -n lia-production

# Check resource usage
kubectl top nodes
kubectl top pods -n lia-production

# View recent logs
kubectl logs -n lia-production -l app=lia-api --tail=100

# Check error rates
curl -s http://prometheus:9090/api/v1/query?query=rate(http_errors_total[5m])
```

#### 2. Review Logs and Alerts

```bash
# Access centralized logs
# Using ELK Stack
curl -X GET "elasticsearch:9200/lia-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{"query": {"match": {"level": "error"}}}'

# Check active alerts
curl -s http://prometheus:9090/api/v1/alerts | jq '.data.alerts'

# Review Grafana dashboards
# Navigate to https://grafana.lia-chatbot.com
```

#### 3. Database Health Check

```sql
-- PostgreSQL health check
SELECT 
    datname,
    numbackends as active_connections,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit
FROM pg_stat_database
WHERE datname = 'lia';

-- Check for long-running queries
SELECT 
    pid,
    now() - query_start as duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
    AND now() - query_start > interval '5 minutes';

-- Check database size
SELECT 
    pg_size_pretty(pg_database_size('lia')) as database_size;
```

```bash
# Redis health check
redis-cli ping
redis-cli info stats
redis-cli info memory

# MongoDB health check
mongosh --eval "db.serverStatus()"
mongosh --eval "db.stats()"
```

### Weekly Tasks

#### 1. Performance Analysis

```bash
# Analyze slow queries (PostgreSQL)
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

# Check API response times
curl -s http://prometheus:9090/api/v1/query \
  --data-urlencode 'query=histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[7d]))'
```

#### 2. Security Audit

```bash
# Review access logs
kubectl logs -n lia-production -l app=lia-api | grep -E "401|403" | tail -100

# Check SSL certificate expiration
echo | openssl s_client -servername api.lia-chatbot.com \
  -connect api.lia-chatbot.com:443 2>/dev/null | \
  openssl x509 -noout -dates

# Review failed authentication attempts
grep "authentication failed" /var/log/lia/access.log | wc -l

# Scan for vulnerabilities
trivy image registry.example.com/lia-api:latest
```

#### 3. Capacity Planning

```bash
# Database growth analysis
SELECT 
    date_trunc('week', created_at) as week,
    count(*) as messages_count,
    pg_size_pretty(sum(length(content))) as total_size
FROM messages
WHERE created_at > now() - interval '3 months'
GROUP BY week
ORDER BY week DESC;

# Disk usage trends
df -h | grep -E "database|redis|elasticsearch"

# Traffic analysis
curl -s http://prometheus:9090/api/v1/query \
  --data-urlencode 'query=rate(http_requests_total[7d])'
```

### Monthly Tasks

#### 1. Backup Verification

```bash
# Verify database backups
aws s3 ls s3://lia-backups/database/ --recursive | tail -10

# Test backup restoration (in test environment)
pg_restore -d lia_test /backups/latest.dump

# Verify Kubernetes backups
velero backup get
velero backup describe lia-production-backup-latest
```

#### 2. Dependency Updates

```bash
# Check for outdated dependencies
npm outdated

# Security audit
npm audit
npm audit fix

# Update dependencies (with testing)
npm update
npm test
```

#### 3. Model Performance Review

```python
# Analyze NLP model performance
import pandas as pd
from sklearn.metrics import classification_report

# Load recent predictions
predictions = load_predictions(days=30)

# Calculate accuracy
accuracy = (predictions['predicted_intent'] == predictions['actual_intent']).mean()
print(f"Intent Classification Accuracy: {accuracy:.2%}")

# Generate classification report
print(classification_report(
    predictions['actual_intent'],
    predictions['predicted_intent']
))

# Identify low-confidence predictions
low_confidence = predictions[predictions['confidence'] < 0.7]
print(f"Low confidence predictions: {len(low_confidence)}")
```

## Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: High Response Time

**Symptoms**:
- API responses taking > 2 seconds
- Users reporting slow chatbot responses

**Diagnosis**:
```bash
# Check application logs
kubectl logs -n lia-production -l app=lia-api --tail=200 | grep -i "slow"

# Check database performance
SELECT * FROM pg_stat_activity WHERE state = 'active';

# Check Redis latency
redis-cli --latency

# Check resource usage
kubectl top pods -n lia-production
```

**Solutions**:
1. **Database Optimization**:
```sql
-- Add missing indexes
CREATE INDEX CONCURRENTLY idx_messages_session_created 
ON messages(session_id, created_at DESC);

-- Vacuum and analyze
VACUUM ANALYZE messages;

-- Update statistics
ANALYZE;
```

2. **Scale Resources**:
```bash
# Increase replicas
kubectl scale deployment/lia-api --replicas=5 -n lia-production

# Increase resource limits
kubectl set resources deployment/lia-api \
  --limits=cpu=1000m,memory=2Gi \
  -n lia-production
```

3. **Enable Caching**:
```yaml
# Update ConfigMap to enable aggressive caching
CACHE_ENABLED: "true"
CACHE_TTL: "600"
RESPONSE_CACHE_ENABLED: "true"
```

#### Issue 2: Memory Leaks

**Symptoms**:
- Pods being OOMKilled
- Memory usage continuously increasing

**Diagnosis**:
```bash
# Monitor memory over time
kubectl top pods -n lia-production --watch

# Get pod details
kubectl describe pod <pod-name> -n lia-production

# Check memory metrics
curl -s http://prometheus:9090/api/v1/query \
  --data-urlencode 'query=container_memory_usage_bytes{pod=~"lia-api.*"}'
```

**Solutions**:
1. **Analyze heap dump** (Node.js):
```bash
# Generate heap snapshot
kubectl exec -n lia-production <pod-name> -- \
  node -e "require('v8').writeHeapSnapshot()"

# Download and analyze with Chrome DevTools
kubectl cp lia-production/<pod-name>:/app/heapsnapshot.heapsnapshot ./
```

2. **Increase memory limits temporarily**:
```bash
kubectl set resources deployment/lia-api \
  --limits=memory=3Gi \
  -n lia-production
```

3. **Implement proper cleanup**:
```javascript
// Example: Clear unused contexts
setInterval(() => {
  contextManager.cleanupExpired();
}, 300000); // Every 5 minutes
```

#### Issue 3: Database Connection Pool Exhausted

**Symptoms**:
- "Too many connections" errors
- Connection timeouts

**Diagnosis**:
```sql
-- Check active connections
SELECT count(*) FROM pg_stat_activity;

-- Check connections by state
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state;

-- Identify connection sources
SELECT 
    usename,
    application_name,
    client_addr,
    count(*)
FROM pg_stat_activity
GROUP BY usename, application_name, client_addr;
```

**Solutions**:
1. **Optimize connection pool**:
```javascript
// Update database configuration
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

2. **Increase max connections** (if appropriate):
```sql
ALTER SYSTEM SET max_connections = 200;
-- Requires PostgreSQL restart
```

3. **Implement connection pooling**:
```bash
# Deploy PgBouncer
kubectl apply -f k8s/pgbouncer-deployment.yaml
```

#### Issue 4: Failed Deployments

**Symptoms**:
- New pods not starting
- Deployment stuck in progress

**Diagnosis**:
```bash
# Check deployment status
kubectl rollout status deployment/lia-api -n lia-production

# Check pod events
kubectl describe pod <failing-pod> -n lia-production

# Check recent logs
kubectl logs <failing-pod> -n lia-production
```

**Solutions**:
1. **Rollback to previous version**:
```bash
kubectl rollout undo deployment/lia-api -n lia-production
```

2. **Fix configuration issues**:
```bash
# Update ConfigMap/Secret
kubectl edit configmap lia-api-config -n lia-production

# Restart deployment
kubectl rollout restart deployment/lia-api -n lia-production
```

3. **Check resource constraints**:
```bash
# Verify node resources
kubectl describe nodes | grep -A 5 "Allocated resources"
```

#### Issue 5: NLP Model Poor Performance

**Symptoms**:
- Low confidence scores
- Incorrect intent classification
- High fallback rate

**Diagnosis**:
```python
# Analyze recent predictions
import pandas as pd

df = load_predictions(days=7)

# Calculate metrics
print(f"Average Confidence: {df['confidence'].mean():.2f}")
print(f"Fallback Rate: {(df['intent'] == 'fallback').mean():.2%}")

# Identify problematic intents
low_conf = df[df['confidence'] < 0.7]
print(low_conf['intent'].value_counts())
```

**Solutions**:
1. **Retrain model with new data**:
```bash
# Prepare training data
python scripts/prepare_training_data.py --days=30

# Train model
python scripts/train_model.py --config=config/nlp/training.yaml

# Evaluate model
python scripts/evaluate_model.py --model=models/latest

# Deploy if metrics improved
python scripts/deploy_model.py --model=models/latest
```

2. **Update intent examples**:
```yaml
# Add more examples to intents.yaml
intents:
  - name: problematic_intent
    examples:
      - "new example 1"
      - "new example 2"
      - "new example 3"
```

3. **Adjust confidence thresholds**:
```yaml
confidence:
  fallback_threshold: 0.35  # Lower to reduce fallback rate
```

## Performance Optimization

### Database Optimization

#### Indexing Strategy

```sql
-- Create composite index for common query
CREATE INDEX CONCURRENTLY idx_messages_user_session_time
ON messages(user_id, session_id, created_at DESC);

-- Create partial index for active sessions
CREATE INDEX CONCURRENTLY idx_active_sessions
ON sessions(status, last_activity)
WHERE status = 'active';

-- Create GIN index for JSONB queries
CREATE INDEX CONCURRENTLY idx_messages_entities_gin
ON messages USING gin(entities jsonb_path_ops);
```

#### Query Optimization

```sql
-- Use EXPLAIN ANALYZE to identify slow queries
EXPLAIN ANALYZE
SELECT m.* FROM messages m
WHERE m.session_id = 'abc123'
ORDER BY m.created_at DESC
LIMIT 50;

-- Optimize with covering index
CREATE INDEX CONCURRENTLY idx_messages_session_coverage
ON messages(session_id, created_at DESC)
INCLUDE (content, intent, confidence_score);
```

#### Vacuum and Maintenance

```bash
# Schedule regular vacuuming
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-vacuum
  namespace: lia-production
spec:
  schedule: "0 2 * * 0"  # Weekly at 2 AM Sunday
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: vacuum
            image: postgres:15
            command:
            - /bin/sh
            - -c
            - |
              PGPASSWORD=\$DB_PASSWORD psql -h \$DB_HOST -U \$DB_USER -d lia \
              -c "VACUUM ANALYZE;"
            env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: lia-api-config
                  key: DB_HOST
          restartPolicy: OnFailure
EOF
```

### Caching Strategy

#### Redis Optimization

```bash
# Configure Redis for optimal performance
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG SET save ""  # Disable RDB if using AOF

# Monitor cache hit rate
redis-cli INFO stats | grep keyspace
```

#### Application-Level Caching

```javascript
// Implement multi-layer caching
class CacheManager {
  constructor() {
    this.memoryCache = new LRUCache({ max: 1000 });
    this.redisClient = createRedisClient();
  }

  async get(key) {
    // Check memory cache first
    let value = this.memoryCache.get(key);
    if (value) return value;

    // Check Redis
    value = await this.redisClient.get(key);
    if (value) {
      this.memoryCache.set(key, value);
      return value;
    }

    return null;
  }

  async set(key, value, ttl = 300) {
    this.memoryCache.set(key, value);
    await this.redisClient.setex(key, ttl, value);
  }
}
```

### Load Balancing

```yaml
# Configure NGINX load balancer
upstream lia_api {
    least_conn;  # Use least connections algorithm
    
    server api-pod-1:8000 max_fails=3 fail_timeout=30s;
    server api-pod-2:8000 max_fails=3 fail_timeout=30s;
    server api-pod-3:8000 max_fails=3 fail_timeout=30s;
    
    keepalive 32;
}

server {
    listen 80;
    server_name api.lia-chatbot.com;
    
    location / {
        proxy_pass http://lia_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Monitoring and Alerts

### Key Metrics to Monitor

```yaml
metrics:
  application:
    - request_rate
    - response_time_p95
    - error_rate
    - active_sessions
    - intent_confidence_avg
    
  infrastructure:
    - cpu_usage
    - memory_usage
    - disk_usage
    - network_io
    
  database:
    - connection_count
    - query_duration
    - transaction_rate
    - replication_lag
    
  business:
    - active_users
    - messages_per_day
    - user_satisfaction_score
    - fallback_rate
```

### Alert Configuration

```yaml
# Prometheus alerts
groups:
  - name: lia_alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "API response time is high"
          
      - alert: DatabaseConnectionsHigh
        expr: pg_stat_database_numbackends{datname="lia"} > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connection count is high"
```

## Data Management

### Data Archival

```sql
-- Archive old messages
CREATE TABLE messages_archive (LIKE messages INCLUDING ALL);

-- Move old data
INSERT INTO messages_archive
SELECT * FROM messages
WHERE created_at < NOW() - INTERVAL '90 days';

-- Delete archived data
DELETE FROM messages
WHERE created_at < NOW() - INTERVAL '90 days';

-- Vacuum to reclaim space
VACUUM FULL messages;
```

### Data Cleanup

```bash
# Automated cleanup job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: data-cleanup
  namespace: lia-production
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: registry.example.com/lia-api:latest
            command:
            - node
            - scripts/cleanup.js
            args:
            - --expired-sessions
            - --old-logs
          restartPolicy: OnFailure
EOF
```

## Security Maintenance

### Certificate Renewal

```bash
# Auto-renewal with cert-manager
kubectl get certificates -n lia-production
kubectl describe certificate lia-api-tls -n lia-production

# Manual renewal if needed
certbot renew --cert-name api.lia-chatbot.com
```

### Secret Rotation

```bash
# Rotate database credentials
# 1. Create new credentials in database
# 2. Update Kubernetes secret
kubectl create secret generic lia-api-secrets \
  --from-literal=DATABASE_URL="new-connection-string" \
  --dry-run=client -o yaml | kubectl apply -f -

# 3. Restart pods to pick up new secret
kubectl rollout restart deployment/lia-api -n lia-production
```

### Security Patches

```bash
# Update base images
docker pull node:18-alpine
docker build -t registry.example.com/lia-api:latest .
docker push registry.example.com/lia-api:latest

# Deploy updated image
kubectl set image deployment/lia-api \
  lia-api=registry.example.com/lia-api:latest \
  -n lia-production
```

## Disaster Recovery Procedures

### Backup Restoration

```bash
# Restore database from backup
pg_restore -h localhost -U lia_user -d lia /backups/lia-backup-2025-10-18.dump

# Restore Kubernetes resources
velero restore create --from-backup lia-production-backup-20251018

# Verify restoration
kubectl get all -n lia-production
```

### Failover Procedures

```bash
# Switch to secondary region
# 1. Update DNS to point to failover region
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch file://failover-dns-update.json

# 2. Promote read replica to primary
aws rds promote-read-replica --db-instance-identifier lia-postgres-replica

# 3. Update application configuration
kubectl set env deployment/lia-api \
  DATABASE_URL="new-primary-connection-string" \
  -n lia-production
```

## Documentation Maintenance

### Keep Documentation Updated

```bash
# Document changes
git add docs/
git commit -m "Update maintenance procedures"
git push origin main

# Update change log
echo "$(date): Updated database optimization procedures" >> docs/CHANGELOG.md
```

## Related Documents

- [Architecture](./architecture.md)
- [DevOps Deployment](./devops-deployment.md)
- [Database Design](./database-design.md)
- [Security](./security.md)
- [API Reference](./api-reference.md)

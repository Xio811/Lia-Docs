# DevOps Deployment Guide

## Overview

This document provides comprehensive guidelines for deploying, managing, and maintaining the LIA chatbot system in production environments using modern DevOps practices.

## Deployment Architecture

### Multi-Environment Strategy

```
Development → Staging → Production

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Development  │ -> │   Staging    │ -> │  Production  │
│              │    │              │    │              │
│ - Feature    │    │ - Integration│    │ - Live Users │
│   Development│    │   Testing    │    │ - High       │
│ - Unit Tests │    │ - Load Tests │    │   Availability│
└──────────────┘    └──────────────┘    └──────────────┘
```

### Infrastructure Overview

```
                    Internet
                        |
                   CloudFlare CDN
                        |
                  Load Balancer
                        |
        ┌───────────────┴───────────────┐
        |                               |
    Web Tier                        API Tier
  (Kubernetes)                    (Kubernetes)
        |                               |
        └───────────────┬───────────────┘
                        |
                  Service Mesh
                        |
        ┌───────────────┴───────────────┐
        |               |               |
    Database         Cache           Search
   (RDS/Azure)     (Redis)      (Elasticsearch)
```

## Container Configuration

### Dockerfile

```dockerfile
# Multi-stage build for LIA API service
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production

# Copy application code
COPY . .

# Build application
RUN npm run build

# Production image
FROM node:18-alpine

# Create non-root user
RUN addgroup -g 1001 -S lia && \
    adduser -S -u 1001 -G lia lia

WORKDIR /app

# Copy built application
COPY --from=builder --chown=lia:lia /app/dist ./dist
COPY --from=builder --chown=lia:lia /app/node_modules ./node_modules
COPY --from=builder --chown=lia:lia /app/package.json ./

# Security: Run as non-root
USER lia

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node healthcheck.js

EXPOSE 8000

CMD ["node", "dist/server.js"]
```

### Docker Compose (Development)

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    depends_on:
      - postgres
      - redis
      - elasticsearch
    volumes:
      - ./src:/app/src
      - ./config:/app/config
    networks:
      - lia-network

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=lia
      - POSTGRES_USER=lia_user
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - lia-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - lia-network

  mongodb:
    image: mongo:6
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    networks:
      - lia-network

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    networks:
      - lia-network

volumes:
  postgres_data:
  redis_data:
  mongo_data:
  es_data:

networks:
  lia-network:
    driver: bridge
```

## Kubernetes Deployment

### Namespace Configuration

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lia-production
  labels:
    name: lia-production
    environment: production
```

### Deployment Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lia-api
  namespace: lia-production
  labels:
    app: lia-api
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: lia-api
  template:
    metadata:
      labels:
        app: lia-api
        version: v1
    spec:
      serviceAccountName: lia-api-sa
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      
      containers:
      - name: lia-api
        image: registry.example.com/lia-api:latest
        imagePullPolicy: Always
        
        ports:
        - containerPort: 8000
          name: http
          protocol: TCP
        
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "8000"
        
        # Secrets from Kubernetes Secrets
        envFrom:
        - secretRef:
            name: lia-api-secrets
        - configMapRef:
            name: lia-api-config
        
        # Resource limits
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Security
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/.cache
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      
      # Pod anti-affinity for high availability
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - lia-api
              topologyKey: kubernetes.io/hostname
```

### Service Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lia-api-service
  namespace: lia-production
  labels:
    app: lia-api
spec:
  type: ClusterIP
  selector:
    app: lia-api
  ports:
  - name: http
    port: 80
    targetPort: 8000
    protocol: TCP
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

### Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lia-api-ingress
  namespace: lia-production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/limit-rps: "10"
spec:
  tls:
  - hosts:
    - api.lia-chatbot.com
    secretName: lia-api-tls
  rules:
  - host: api.lia-chatbot.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: lia-api-service
            port:
              number: 80
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lia-api-config
  namespace: lia-production
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  TIMEOUT: "30000"
  CORS_ORIGIN: "https://app.lia-chatbot.com"
```

### Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: lia-api-secrets
  namespace: lia-production
type: Opaque
data:
  DATABASE_URL: <base64-encoded>
  JWT_SECRET: <base64-encoded>
  API_KEY: <base64-encoded>
```

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: lia-api-hpa
  namespace: lia-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lia-api
  minReplicas: 3
  maxReplicas: 10
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Run security scan
        run: npm audit

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}
      
      - name: Deploy to staging
        run: |
          kubectl set image deployment/lia-api \
            lia-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:develop \
            -n lia-staging
          kubectl rollout status deployment/lia-api -n lia-staging

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG_PROD }}
      
      - name: Deploy to production
        run: |
          kubectl set image deployment/lia-api \
            lia-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main \
            -n lia-production
          kubectl rollout status deployment/lia-api -n lia-production
      
      - name: Verify deployment
        run: |
          kubectl get pods -n lia-production
          kubectl get services -n lia-production
```

## Infrastructure as Code

### Terraform Configuration

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "lia-terraform-state"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "lia-vpc"
    Environment = var.environment
  }
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "lia-cluster"
  role_arn = aws_iam_role.cluster.arn
  version  = "1.28"
  
  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true
    endpoint_public_access  = true
  }
  
  depends_on = [
    aws_iam_role_policy_attachment.cluster_policy
  ]
}

# RDS Database
resource "aws_db_instance" "postgres" {
  identifier        = "lia-postgres"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  storage_encrypted = true
  
  db_name  = "lia"
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.database.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = false
  final_snapshot_identifier = "lia-postgres-final-snapshot"
  
  tags = {
    Name        = "lia-postgres"
    Environment = var.environment
  }
}

# ElastiCache Redis
resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "lia-redis"
  engine              = "redis"
  engine_version      = "7.0"
  node_type           = "cache.t3.medium"
  num_cache_nodes     = 1
  parameter_group_name = "default.redis7"
  port                = 6379
  
  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.redis.id]
  
  tags = {
    Name        = "lia-redis"
    Environment = var.environment
  }
}
```

## Monitoring and Observability

### Prometheus Configuration

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'lia-api'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - lia-production
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: lia-api
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "title": "LIA Chatbot Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Response Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
          }
        ]
      }
    ]
  }
}
```

## Backup and Disaster Recovery

### Backup Strategy

```yaml
backup:
  database:
    type: automated
    schedule: "0 2 * * *"  # Daily at 2 AM
    retention: 30 days
    location: s3://lia-backups/database/
    
  kubernetes:
    tool: Velero
    schedule: "0 3 * * *"  # Daily at 3 AM
    retention: 14 days
    include_namespaces:
      - lia-production
    
  configuration:
    type: git
    repository: github.com/company/lia-config
    backup_schedule: on_change
```

### Disaster Recovery Plan

```yaml
disaster_recovery:
  rto: 4 hours  # Recovery Time Objective
  rpo: 1 hour   # Recovery Point Objective
  
  procedures:
    - assess_damage
    - notify_stakeholders
    - failover_to_backup_region
    - restore_from_backup
    - verify_functionality
    - update_dns
    - monitor_recovery
  
  testing:
    frequency: quarterly
    type: full_simulation
```

## Scaling Strategy

### Vertical Scaling

```yaml
vertical_scaling:
  triggers:
    - metric: cpu_usage
      threshold: 80%
      action: increase_resources
    - metric: memory_usage
      threshold: 85%
      action: increase_resources
  
  resource_tiers:
    - tier: small
      cpu: "250m"
      memory: "512Mi"
    - tier: medium
      cpu: "500m"
      memory: "1Gi"
    - tier: large
      cpu: "1000m"
      memory: "2Gi"
```

### Horizontal Scaling

```yaml
horizontal_scaling:
  metrics:
    - type: cpu
      threshold: 70%
    - type: memory
      threshold: 80%
    - type: custom
      metric: requests_per_second
      threshold: 1000
  
  scaling_behavior:
    scale_up:
      speed: fast
      cooldown: 60s
    scale_down:
      speed: slow
      cooldown: 300s
```

## Rollback Procedures

```bash
# Rollback to previous deployment
kubectl rollout undo deployment/lia-api -n lia-production

# Rollback to specific revision
kubectl rollout undo deployment/lia-api --to-revision=2 -n lia-production

# Check rollout history
kubectl rollout history deployment/lia-api -n lia-production

# Verify rollback status
kubectl rollout status deployment/lia-api -n lia-production
```

## Health Checks

### Application Health Check

```javascript
// healthcheck.js
const http = require('http');

const options = {
  host: 'localhost',
  port: 8000,
  path: '/health',
  timeout: 2000
};

const request = http.request(options, (res) => {
  if (res.statusCode === 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

request.on('error', (err) => {
  process.exit(1);
});

request.end();
```

## Related Documents

- [Architecture](./architecture.md)
- [Security](./security.md)
- [Maintenance Guide](./maintenance-guide.md)
- [Database Design](./database-design.md)

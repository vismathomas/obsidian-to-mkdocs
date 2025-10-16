# Deployment Guide

Production deployment strategies for Fjas applications.

## Table of Contents

- [Deployment Options](#deployment-options)
- [Docker Deployment](#docker-deployment)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Cloud Platforms](#cloud-platforms)
- [Monitoring & Logging](#monitoring--logging)
- [CI/CD Pipeline](#cicd-pipeline)

## Deployment Options

Choose the deployment strategy that fits your needs:

```mermaid
graph TD
    Start[Choose Deployment]
    
    Start --> Q1{Container-based?}
    Q1 -->|Yes| Docker[Docker]
    Q1 -->|No| Traditional[Traditional VM/Server]
    
    Docker --> Q2{Orchestration needed?}
    Q2 -->|Yes| K8s[Kubernetes]
    Q2 -->|No| DockerCompose[Docker Compose]
    
    Traditional --> Q3{Cloud provider?}
    Q3 -->|AWS| EC2[EC2 + ECS/EKS]
    Q3 -->|Azure| Azure[Azure App Service]
    Q3 -->|GCP| GCP[GCP App Engine/GKE]
    Q3 -->|Self-hosted| SelfHost[VPS/Dedicated]
```

## Docker Deployment

### Dockerfile

Create an optimized `Dockerfile`:

```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application files
COPY . .

# Build application
RUN npm run build

# Production image
FROM node:18-alpine

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create app user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy from builder
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "dist/index.js"]
```

### .dockerignore

```
node_modules
npm-debug.log
.git
.gitignore
.env
.env.local
dist
coverage
*.md
.vscode
.idea
```

### Build and Run

```bash
# Build image
docker build -t fjas-api:latest .

# Run container
docker run -d \
  --name fjas-api \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -e REDIS_HOST=redis \
  --restart unless-stopped \
  fjas-api:latest

# View logs
docker logs -f fjas-api

# Stop container
docker stop fjas-api
```

### Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - REDIS_HOST=redis
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/myapp
    depends_on:
      - redis
      - postgres
    restart: unless-stopped
    networks:
      - app-network
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped
    networks:
      - app-network

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
    restart: unless-stopped
    networks:
      - app-network

volumes:
  redis-data:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

### Nginx Configuration

```nginx
upstream fjas_backend {
    least_conn;
    server api:3000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name api.example.com;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    # SSL certificates
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript;
    
    # Client body size
    client_max_body_size 10M;
    
    location / {
        proxy_pass http://fjas_backend;
        proxy_http_version 1.1;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffers
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        proxy_pass http://fjas_backend/health;
    }
}
```

## Kubernetes Deployment

### Deployment Manifest

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fjas-api
  labels:
    app: fjas-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fjas-api
  template:
    metadata:
      labels:
        app: fjas-api
    spec:
      containers:
      - name: fjas
        image: your-registry/fjas-api:latest
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          value: "production"
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: fjas-config
              key: redis.host
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: fjas-secrets
              key: database.url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
      imagePullSecrets:
      - name: registry-credentials
---
apiVersion: v1
kind: Service
metadata:
  name: fjas-api-service
spec:
  selector:
    app: fjas-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fjas-config
data:
  redis.host: "redis-service"
  redis.port: "6379"
---
apiVersion: v1
kind: Secret
metadata:
  name: fjas-secrets
type: Opaque
stringData:
  database.url: "postgresql://user:pass@postgres:5432/myapp"
  jwt.secret: "your-jwt-secret"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fjas-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fjas-api
  minReplicas: 3
  maxReplicas: 20
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

### Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fjas-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: fjas-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fjas-api-service
            port:
              number: 80
```

### Deploy to Kubernetes

```bash
# Apply configurations
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/ingress.yaml

# Check status
kubectl get pods -l app=fjas-api
kubectl get svc fjas-api-service
kubectl get hpa fjas-hpa

# View logs
kubectl logs -l app=fjas-api -f

# Scale manually
kubectl scale deployment fjas-api --replicas=5

# Update image
kubectl set image deployment/fjas-api fjas=your-registry/fjas-api:v2.0.0

# Rollback
kubectl rollout undo deployment/fjas-api
```

## Cloud Platforms

### AWS Elastic Beanstalk

```yaml
# .ebextensions/01-node.config
option_settings:
  aws:elasticbeanstalk:container:nodejs:
    NodeCommand: "npm start"
    NodeVersion: 18.x
  aws:elasticbeanstalk:environment:proxy:
    ProxyServer: nginx
  aws:autoscaling:launchconfiguration:
    InstanceType: t3.medium
  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 10
```

### Google Cloud Run

```bash
# Build and deploy
gcloud builds submit --tag gcr.io/PROJECT_ID/fjas-api

gcloud run deploy fjas-api \
  --image gcr.io/PROJECT_ID/fjas-api \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 1 \
  --max-instances 10 \
  --set-env-vars NODE_ENV=production
```

### Azure Container Instances

```bash
az container create \
  --resource-group myResourceGroup \
  --name fjas-api \
  --image your-registry.azurecr.io/fjas-api:latest \
  --dns-name-label fjas-api \
  --ports 3000 \
  --cpu 1 \
  --memory 1 \
  --environment-variables \
    NODE_ENV=production \
    REDIS_HOST=redis.example.com
```

## Monitoring & Logging

### Prometheus + Grafana

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'fjas-api'
    static_configs:
      - targets: ['fjas-api:9090']
    metrics_path: /metrics
```

### ELK Stack (Elasticsearch, Logstash, Kibana)

```javascript
const winston = require('winston');
const { ElasticsearchTransport } = require('winston-elasticsearch');

const logger = winston.createLogger({
  transports: [
    new ElasticsearchTransport({
      level: 'info',
      clientOpts: {
        node: 'http://elasticsearch:9200'
      },
      index: 'fjas-logs'
    })
  ]
});
```

## CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Run linter
        run: npm run lint

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker image
        run: docker build -t fjas-api:${{ github.sha }} .
      
      - name: Push to registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker tag fjas-api:${{ github.sha }} your-registry/fjas-api:latest
          docker push your-registry/fjas-api:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v1
        with:
          manifests: |
            k8s/deployment.yaml
          images: |
            your-registry/fjas-api:latest
          kubectl-version: 'latest'
```

## Zero-Downtime Deployment

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant V1 as Version 1.0
    participant V2 as Version 2.0
    
    Note over V1: Running (3 replicas)
    
    LB->>V1: Route all traffic
    
    Note over V2: Deploy new version
    V2->>V2: Start replica 1
    V2->>V2: Health check passes
    
    LB->>V2: Route some traffic
    LB->>V1: Route remaining traffic
    
    V2->>V2: Start replica 2
    V2->>V2: Health check passes
    
    LB->>V2: Route more traffic
    LB->>V1: Route less traffic
    
    V2->>V2: Start replica 3
    V2->>V2: Health check passes
    
    LB->>V2: Route all traffic
    
    Note over V1: Terminate old version
```

## Next Steps

- [Troubleshooting](troubleshooting.md) - Debug production issues
- [Performance Tuning](performance.md) - Optimize for production
- [Architecture](architecture.md) - Understand system design

---

[← Back to Documentation](../README.md#documentation)

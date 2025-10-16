# Architecture Overview

This document provides a comprehensive overview of Fjas architecture, design principles, and internal components.

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [High-Level Architecture](#high-level-architecture)
- [Core Components](#core-components)
- [Request Processing Pipeline](#request-processing-pipeline)
- [Caching Strategy](#caching-strategy)
- [Scalability Patterns](#scalability-patterns)

## Design Philosophy

Fjas is built on four core principles:

1. **Performance First** - Sub-millisecond response times
2. **Developer Experience** - Intuitive APIs and excellent tooling
3. **Production Ready** - Battle-tested with observability built-in
4. **Cloud Native** - Designed for containers and orchestration

## High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        WebApp[Web Application]
        Mobile[Mobile App]
        IoT[IoT Devices]
    end

    subgraph "Edge Layer"
        CDN[CDN / Edge Cache]
        WAF[Web Application Firewall]
    end

    subgraph "API Gateway Layer"
        LB[Load Balancer]
        
        subgraph "Fjas Cluster"
            F1[Fjas Node 1]
            F2[Fjas Node 2]
            F3[Fjas Node 3]
        end
    end

    subgraph "Cache Layer"
        Redis[(Redis Cluster)]
        Memcached[(Memcached)]
    end

    subgraph "Service Layer"
        Auth[Auth Service]
        User[User Service]
        Product[Product Service]
        Order[Order Service]
    end

    subgraph "Data Layer"
        PG[(PostgreSQL)]
        Mongo[(MongoDB)]
        S3[(Object Storage)]
    end

    subgraph "Observability"
        Metrics[Prometheus]
        Traces[Jaeger]
        Logs[Elasticsearch]
    end

    WebApp --> CDN
    Mobile --> CDN
    IoT --> CDN
    
    CDN --> WAF
    WAF --> LB
    
    LB --> F1
    LB --> F2
    LB --> F3
    
    F1 --> Redis
    F2 --> Redis
    F3 --> Redis
    
    F1 --> Auth
    F1 --> User
    F1 --> Product
    F1 --> Order
    
    Auth --> PG
    User --> PG
    Product --> Mongo
    Order --> PG
    Order --> S3
    
    F1 -.-> Metrics
    F1 -.-> Traces
    F1 -.-> Logs
```

## Core Components

### 1. HTTP Server

The HTTP server is built on top of Node.js's native `http` module with custom optimizations:

```typescript
interface ServerConfig {
  host: string;
  port: number;
  workers: number;
  keepAliveTimeout: number;
  headersTimeout: number;
  maxHeaderSize: number;
}

class FjasServer {
  private server: http.Server;
  private router: Router;
  private middleware: Middleware[];
  
  constructor(config: ServerConfig) {
    this.server = http.createServer(this.handleRequest.bind(this));
    this.configureOptimizations(config);
  }
  
  private async handleRequest(req: Request, res: Response) {
    const context = this.createContext(req, res);
    await this.middleware.run(context);
    await this.router.handle(context);
  }
}
```

### 2. Router

Fast routing using radix tree algorithm:

```mermaid
graph TD
    Root["/"]
    Root --> Users["/users"]
    Root --> Products["/products"]
    Root --> Orders["/orders"]
    
    Users --> UsersId["/users/:id"]
    UsersId --> UsersPosts["/users/:id/posts"]
    
    Products --> ProductsId["/products/:id"]
    Products --> ProductsSearch["/products/search"]
    
    Orders --> OrdersId["/orders/:id"]
    OrdersId --> OrdersCancel["/orders/:id/cancel"]
```

**Routing Performance:**

- O(k) lookup time where k is the URL length
- Supports parameter matching
- Wildcard patterns
- Method-based routing

### 3. Middleware Engine

Middleware are executed in a chain with support for async operations:

```javascript
class MiddlewareEngine {
  constructor() {
    this.stack = [];
  }
  
  use(fn) {
    this.stack.push(fn);
  }
  
  async run(context, index = 0) {
    if (index >= this.stack.length) return;
    
    const middleware = this.stack[index];
    await middleware(context, async () => {
      await this.run(context, index + 1);
    });
  }
}
```

**Built-in Middleware:**

| Middleware | Purpose | Order |
|------------|---------|-------|
| Logger | Request/response logging | 1 |
| CORS | Cross-origin resource sharing | 2 |
| Helmet | Security headers | 3 |
| Rate Limiter | Request throttling | 4 |
| Body Parser | Parse request body | 5 |
| Authentication | Verify credentials | 6 |
| Compression | Gzip/Brotli compression | 7 |

### 4. Schema Validator

JSON Schema validation with JIT compilation:

```javascript
const Ajv = require('ajv');

class SchemaValidator {
  constructor() {
    this.ajv = new Ajv({
      coerceTypes: true,
      useDefaults: true,
      removeAdditional: true,
      allErrors: false
    });
    this.compiled = new Map();
  }
  
  compile(schema) {
    const key = JSON.stringify(schema);
    if (!this.compiled.has(key)) {
      this.compiled.set(key, this.ajv.compile(schema));
    }
    return this.compiled.get(key);
  }
  
  validate(data, schema) {
    const validate = this.compile(schema);
    const valid = validate(data);
    
    if (!valid) {
      throw new ValidationError(validate.errors);
    }
    
    return data;
  }
}
```

### 5. Cache Manager

Multi-layer caching with automatic invalidation:

```mermaid
graph LR
    Request[Request]
    L1[L1: Memory Cache]
    L2[L2: Redis Cache]
    Backend[Backend Service]
    
    Request --> L1
    L1 -->|Miss| L2
    L2 -->|Miss| Backend
    Backend -->|Store| L2
    L2 -->|Store| L1
    L1 -->|Response| Request
```

**Cache Implementation:**

```javascript
class CacheManager {
  constructor(config) {
    this.memory = new LRUCache({ max: 1000 });
    this.redis = new Redis(config.redis);
  }
  
  async get(key) {
    // Try L1 (memory) first
    let value = this.memory.get(key);
    if (value) return value;
    
    // Try L2 (Redis)
    value = await this.redis.get(key);
    if (value) {
      this.memory.set(key, value);
      return value;
    }
    
    return null;
  }
  
  async set(key, value, ttl) {
    this.memory.set(key, value);
    await this.redis.setex(key, ttl, value);
  }
  
  async invalidate(pattern) {
    // Clear matching keys from both layers
    this.memory.clear();
    const keys = await this.redis.keys(pattern);
    if (keys.length) {
      await this.redis.del(...keys);
    }
  }
}
```

## Request Processing Pipeline

The journey of an HTTP request through Fjas:

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant LoadBalancer
    participant Fjas
    participant Middleware
    participant Cache
    participant Validator
    participant Handler
    participant Service
    
    Client->>LoadBalancer: HTTP Request
    LoadBalancer->>Fjas: Route to instance
    Fjas->>Middleware: Logger
    Fjas->>Middleware: CORS
    Fjas->>Middleware: Rate Limiter
    
    alt Rate limit exceeded
        Middleware-->>Client: 429 Too Many Requests
    end
    
    Fjas->>Cache: Check cache
    
    alt Cache hit
        Cache-->>Client: Cached response
    else Cache miss
        Fjas->>Validator: Validate request
        
        alt Validation failed
            Validator-->>Client: 400 Bad Request
        end
        
        Fjas->>Handler: Execute handler
        Handler->>Service: Call backend service
        Service-->>Handler: Response data
        Handler->>Cache: Store in cache
        Handler-->>Client: JSON Response
    end
```

### Performance Metrics

| Stage | Time | % of Total |
|-------|------|------------|
| Network I/O | 0.1ms | 12% |
| Middleware | 0.2ms | 25% |
| Cache lookup | 0.1ms | 12% |
| Validation | 0.15ms | 19% |
| Handler execution | 0.25ms | 31% |
| Serialization | 0.05ms | 6% |
| **Total** | **0.8ms** | **100%** |

## Caching Strategy

### Cache Key Generation

```javascript
function generateCacheKey(req) {
  const parts = [
    req.method,
    req.url,
    req.headers['accept-language'] || 'en',
    req.user?.id || 'anonymous'
  ];
  return crypto
    .createHash('sha256')
    .update(parts.join(':'))
    .digest('hex')
    .slice(0, 16);
}
```

### Cache Invalidation Patterns

```mermaid
graph TD
    Event[Data Update Event]
    Event --> TagBased[Tag-Based Invalidation]
    Event --> TimeBased[Time-Based Expiry]
    Event --> EventBased[Event-Driven Purge]
    
    TagBased --> InvalidateTags[Invalidate all keys with tag]
    TimeBased --> SetTTL[Set TTL on cache entry]
    EventBased --> PurgePattern[Purge by pattern match]
```

### Cache Policies

1. **Cache-Control Headers**
   ```http
   Cache-Control: public, max-age=300, s-maxage=600
   ETag: "33a64df551425fcc55e4d42a148795d9"
   Vary: Accept-Encoding, Accept-Language
   ```

2. **Conditional Requests**
   ```javascript
   if (req.headers['if-none-match'] === etag) {
     return res.status(304).end(); // Not Modified
   }
   ```

3. **Cache Warming**
   ```javascript
   // Preload frequently accessed data
   async function warmCache() {
     const popularEndpoints = ['/products', '/categories'];
     for (const endpoint of popularEndpoints) {
       const data = await fetchData(endpoint);
       await cache.set(endpoint, data, 3600);
     }
   }
   ```

## Scalability Patterns

### Horizontal Scaling

Fjas instances are stateless and can be scaled horizontally:

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fjas-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fjas
  template:
    metadata:
      labels:
        app: fjas
    spec:
      containers:
      - name: fjas
        image: fjas/fjas:2.1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: REDIS_HOST
          value: "redis-service"
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

### Load Balancing Strategies

```mermaid
graph TB
    Client[Clients]
    
    subgraph "Load Balancing Algorithms"
        RR[Round Robin]
        LC[Least Connections]
        IPH[IP Hash]
        WRR[Weighted Round Robin]
    end
    
    subgraph "Health Checks"
        TCP[TCP Check]
        HTTP[HTTP Check]
        Custom[Custom Script]
    end
    
    Client --> RR
    Client --> LC
    Client --> IPH
    Client --> WRR
    
    RR --> TCP
    LC --> HTTP
    IPH --> Custom
    WRR --> HTTP
```

### Circuit Breaker Pattern

Protect backend services from cascading failures:

```javascript
class CircuitBreaker {
  constructor(options) {
    this.threshold = options.threshold || 5;
    this.timeout = options.timeout || 60000;
    this.state = 'CLOSED';
    this.failures = 0;
    this.nextAttempt = Date.now();
  }
  
  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failures++;
    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}
```

## Security Architecture

### Defense in Depth

```mermaid
graph TD
    Request[Incoming Request]
    
    Request --> L1[Layer 1: Network Security]
    L1 --> Firewall[Firewall Rules]
    L1 --> DDoS[DDoS Protection]
    
    Firewall --> L2[Layer 2: Application Security]
    L2 --> WAF[Web Application Firewall]
    L2 --> RateLimit[Rate Limiting]
    
    WAF --> L3[Layer 3: Authentication]
    L3 --> JWT[JWT Validation]
    L3 --> OAuth[OAuth2 / OIDC]
    
    JWT --> L4[Layer 4: Authorization]
    L4 --> RBAC[Role-Based Access Control]
    L4 --> ABAC[Attribute-Based Access Control]
    
    RBAC --> L5[Layer 5: Data Security]
    L5 --> Encrypt[Encryption at Rest]
    L5 --> TLS[TLS in Transit]
    
    TLS --> Backend[Backend Services]
```

## Monitoring & Observability

### OpenTelemetry Integration

```javascript
const { trace, metrics } = require('@opentelemetry/api');

// Tracing
const tracer = trace.getTracer('fjas');

async function handleRequest(req, res) {
  const span = tracer.startSpan('http.request', {
    attributes: {
      'http.method': req.method,
      'http.url': req.url,
      'http.user_agent': req.headers['user-agent']
    }
  });
  
  try {
    const result = await processRequest(req);
    span.setStatus({ code: SpanStatusCode.OK });
    return result;
  } catch (error) {
    span.setStatus({ 
      code: SpanStatusCode.ERROR,
      message: error.message 
    });
    throw error;
  } finally {
    span.end();
  }
}

// Metrics
const requestCounter = metrics.createCounter('http.requests', {
  description: 'Total number of HTTP requests'
});

const latencyHistogram = metrics.createHistogram('http.request.duration', {
  description: 'HTTP request duration in milliseconds'
});
```

## Next Steps

- [Configuration Guide](configuration.md) - Configure Fjas for your needs
- [API Reference](api-reference.md) - Detailed API documentation
- [Performance Tuning](performance.md) - Optimize performance
- [Deployment Guide](deployment.md) - Deploy to production

---

[← Back to Documentation](../README.md#documentation)

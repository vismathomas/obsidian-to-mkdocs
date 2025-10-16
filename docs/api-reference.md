# API Reference

Complete API reference for Fjas Framework.

## Table of Contents

- [Core API](#core-api)
- [Router API](#router-api)
- [Middleware API](#middleware-api)
- [Request Object](#request-object)
- [Response Object](#response-object)
- [Schema Validation](#schema-validation)
- [Cache API](#cache-api)
- [Configuration](#configuration)
- [LICENSE](LICENSE)

## Core API

### Fjas Class

The main class for creating a Fjas application.

#### Constructor

```typescript
new Fjas(options?: FjasOptions): Fjas
```

**Parameters:**

```typescript
interface FjasOptions {
  port?: number;              // Server port (default: 3000)
  host?: string;              // Server host (default: '0.0.0.0')
  workers?: number;           // Number of worker processes (default: CPU count)
  cache?: CacheOptions;       // Cache configuration
  logging?: LoggingOptions;   // Logging configuration
  security?: SecurityOptions; // Security settings
}
```

**Example:**

```javascript
const Fjas = require('fjas');

const app = new Fjas({
  port: 3000,
  workers: 4,
  cache: {
    enabled: true,
    provider: 'redis',
    ttl: 300
  }
});
```

#### Methods

##### `app.route(path, options)`

Define a route handler.

```typescript
route(path: string, options: RouteOptions): void
```

**Example:**

```javascript
app.route('/users/:id', {
  method: 'GET',
  schema: {
    params: {
      type: 'object',
      properties: {
        id: { type: 'number' }
      }
    }
  },
  handler: async (req, res) => {
    const user = await User.findById(req.params.id);
    return res.json(user);
  }
});
```

##### `app.use(middleware)`

Add middleware to the application.

```typescript
use(middleware: Middleware | Middleware[]): void
```

**Example:**

```javascript
// Single middleware
app.use(async (req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  await next();
});

// Multiple middleware
app.use([
  cors(),
  helmet(),
  bodyParser.json()
]);
```

##### `app.start()`

Start the server.

```typescript
async start(): Promise<void>
```

**Example:**

```javascript
await app.start();
console.log('Server started on port 3000');
```

##### `app.stop()`

Gracefully stop the server.

```typescript
async stop(): Promise<void>
```

**Example:**

```javascript
process.on('SIGTERM', async () => {
  await app.stop();
  process.exit(0);
});
```

## Router API

### Router Class

Create modular route handlers.

```typescript
const { Router } = require('fjas');
const router = new Router();
```

#### HTTP Methods

```typescript
router.get(path: string, options: RouteOptions): void
router.post(path: string, options: RouteOptions): void
router.put(path: string, options: RouteOptions): void
router.patch(path: string, options: RouteOptions): void
router.delete(path: string, options: RouteOptions): void
router.options(path: string, options: RouteOptions): void
router.head(path: string, options: RouteOptions): void
```

**Example:**

```javascript
// GET /api/users
router.get('/users', {
  description: 'List all users',
  handler: async (req, res) => {
    const users = await User.findAll();
    return res.json(users);
  }
});

// POST /api/users
router.post('/users', {
  description: 'Create new user',
  schema: {
    body: {
      type: 'object',
      required: ['name', 'email'],
      properties: {
        name: { type: 'string', minLength: 2 },
        email: { type: 'string', format: 'email' }
      }
    }
  },
  handler: async (req, res) => {
    const user = await User.create(req.body);
    return res.status(201).json(user);
  }
});
```

#### Route Parameters

```mermaid
graph LR
    A["/users/:id/posts/:postId"] --> B[Path Parameters]
    B --> C["req.params.id"]
    B --> D["req.params.postId"]
    
    E["/search?q=fjas&limit=10"] --> F[Query Parameters]
    F --> G["req.query.q"]
    F --> H["req.query.limit"]
```

**Example:**

```javascript
// Path parameters
router.get('/users/:userId/posts/:postId', {
  handler: async (req, res) => {
    const { userId, postId } = req.params;
    const post = await Post.findOne({ 
      userId, 
      id: postId 
    });
    return res.json(post);
  }
});

// Query parameters
router.get('/search', {
  schema: {
    query: {
      type: 'object',
      properties: {
        q: { type: 'string', minLength: 1 },
        limit: { type: 'number', default: 10, maximum: 100 },
        offset: { type: 'number', default: 0 }
      },
      required: ['q']
    }
  },
  handler: async (req, res) => {
    const { q, limit, offset } = req.query;
    const results = await search(q, { limit, offset });
    return res.json(results);
  }
});
```

## Middleware API

### Creating Middleware

```typescript
type Middleware = (
  req: Request,
  res: Response,
  next: NextFunction
) => Promise<void> | void;
```

**Basic Middleware:**

```javascript
const logger = async (req, res, next) => {
  const start = Date.now();
  
  await next(); // Call next middleware
  
  const duration = Date.now() - start;
  console.log(`${req.method} ${req.url} - ${duration}ms`);
};

app.use(logger);
```

**Error Handling Middleware:**

```javascript
const errorHandler = async (req, res, next) => {
  try {
    await next();
  } catch (error) {
    console.error(error);
    res.status(error.statusCode || 500).json({
      error: error.message,
      stack: process.env.NODE_ENV === 'development' ? error.stack : undefined
    });
  }
};

app.use(errorHandler);
```

### Built-in Middleware

#### CORS

```javascript
const { cors } = require('fjas/middleware');

app.use(cors({
  origins: ['https://example.com', 'https://app.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Total-Count'],
  credentials: true,
  maxAge: 86400
}));
```

#### Rate Limiting

```javascript
const { rateLimit } = require('fjas/middleware');

app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  message: 'Too many requests, please try again later',
  statusCode: 429,
  skipSuccessfulRequests: false,
  keyGenerator: (req) => req.ip
}));
```

#### Authentication

```javascript
const { auth } = require('fjas/middleware');

// JWT authentication
const jwtAuth = auth.jwt({
  secret: process.env.JWT_SECRET,
  algorithms: ['HS256'],
  getToken: (req) => {
    const header = req.headers.authorization;
    if (header && header.startsWith('Bearer ')) {
      return header.substring(7);
    }
    return null;
  }
});

// API Key authentication
const apiKeyAuth = auth.apiKey({
  header: 'X-API-Key',
  validate: async (key) => {
    return await ApiKey.verify(key);
  }
});

// Apply to routes
router.get('/protected', {
  middleware: [jwtAuth],
  handler: async (req, res) => {
    return res.json({ user: req.user });
  }
});
```

## Request Object

The request object represents the HTTP request.

### Properties

```typescript
interface Request {
  method: string;              // HTTP method (GET, POST, etc.)
  url: string;                 // Request URL
  path: string;                // URL path
  query: Record<string, any>;  // Query parameters
  params: Record<string, any>; // Path parameters
  headers: Record<string, string>; // Request headers
  body: any;                   // Parsed request body
  cookies: Record<string, string>; // Parsed cookies
  ip: string;                  // Client IP address
  user?: any;                  // Authenticated user (set by auth middleware)
}
```

### Methods

#### `req.get(header)`

Get request header value.

```javascript
const userAgent = req.get('user-agent');
const contentType = req.get('content-type');
```

#### `req.is(type)`

Check if request is of given type.

```javascript
if (req.is('json')) {
  // Handle JSON request
}

if (req.is('multipart/form-data')) {
  // Handle file upload
}
```

## Response Object

The response object represents the HTTP response.

### Methods

#### `res.status(code)`

Set HTTP status code.

```javascript
res.status(404).json({ error: 'Not found' });
res.status(201).json({ id: newUser.id });
```

#### `res.json(data)`

Send JSON response.

```javascript
res.json({ message: 'Success', data: users });
res.json({ error: 'Invalid input' });
```

#### `res.send(data)`

Send response (auto-detects content type).

```javascript
res.send('Hello World');
res.send({ message: 'Success' });
res.send(Buffer.from('binary data'));
```

#### `res.set(header, value)`

Set response header.

```javascript
res.set('Content-Type', 'application/json');
res.set('Cache-Control', 'public, max-age=300');
res.set('X-Custom-Header', 'value');
```

#### `res.cookie(name, value, options)`

Set cookie.

```javascript
res.cookie('sessionId', '123abc', {
  httpOnly: true,
  secure: true,
  maxAge: 3600000, // 1 hour
  sameSite: 'strict'
});
```

#### `res.redirect(url, status?)`

Redirect to URL.

```javascript
res.redirect('/login');
res.redirect('https://example.com', 301);
```

## Schema Validation

Fjas uses JSON Schema for request validation.

### Schema Structure

```typescript
interface RouteSchema {
  params?: JSONSchema;  // Path parameters
  query?: JSONSchema;   // Query parameters
  body?: JSONSchema;    // Request body
  headers?: JSONSchema; // Request headers
  response?: {          // Response schema
    [statusCode: number]: JSONSchema;
  };
}
```

### Example: Complete Validation

```javascript
router.post('/users', {
  schema: {
    body: {
      type: 'object',
      required: ['name', 'email', 'password'],
      properties: {
        name: { 
          type: 'string', 
          minLength: 2,
          maxLength: 50
        },
        email: { 
          type: 'string', 
          format: 'email'
        },
        password: { 
          type: 'string', 
          minLength: 8,
          pattern: '^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)'
        },
        age: { 
          type: 'number', 
          minimum: 18,
          maximum: 120
        },
        role: {
          type: 'string',
          enum: ['user', 'admin', 'moderator']
        }
      }
    },
    response: {
      201: {
        type: 'object',
        properties: {
          id: { type: 'number' },
          name: { type: 'string' },
          email: { type: 'string' },
          createdAt: { type: 'string', format: 'date-time' }
        }
      }
    }
  },
  handler: async (req, res) => {
    const user = await User.create(req.body);
    return res.status(201).json(user);
  }
});
```

### Custom Validators

```javascript
const { ajv } = require('fjas/validation');

// Add custom format
ajv.addFormat('phone', {
  validate: (data) => /^\+?[1-9]\d{1,14}$/.test(data)
});

// Add custom keyword
ajv.addKeyword('isUnique', {
  async: true,
  validate: async (schema, data) => {
    const exists = await User.findOne({ email: data });
    return !exists;
  }
});

// Use in schema
router.post('/users', {
  schema: {
    body: {
      type: 'object',
      properties: {
        phone: { type: 'string', format: 'phone' },
        email: { type: 'string', isUnique: true }
      }
    }
  },
  handler: async (req, res) => {
    // Handler code
  }
});
```

## Cache API

### Cache Interface

```typescript
interface Cache {
  get(key: string): Promise<any>;
  set(key: string, value: any, ttl?: number): Promise<void>;
  del(key: string): Promise<void>;
  clear(): Promise<void>;
  keys(pattern: string): Promise<string[]>;
}
```

### Usage Examples

```javascript
const cache = app.cache;

// Get value
const user = await cache.get('user:123');

// Set value with TTL (seconds)
await cache.set('user:123', userData, 3600);

// Delete value
await cache.del('user:123');

// Clear all cache
await cache.clear();

// Find keys by pattern
const userKeys = await cache.keys('user:*');
```

### Route-Level Caching

```javascript
router.get('/users/:id', {
  cache: {
    enabled: true,
    ttl: 300, // 5 minutes
    key: (req) => `user:${req.params.id}`,
    invalidate: ['user:created', 'user:updated']
  },
  handler: async (req, res) => {
    const user = await User.findById(req.params.id);
    return res.json(user);
  }
});
```

### Cache Tags

```javascript
// Set cache with tags
await cache.set('product:123', product, 3600, {
  tags: ['products', 'electronics']
});

// Invalidate by tag
await cache.invalidateTag('products');
```

## Configuration

### Configuration File

Create `fjas.config.yml`:

```yaml
# Server Configuration
server:
  host: 0.0.0.0
  port: 3000
  workers: 4
  keepAliveTimeout: 65000
  headersTimeout: 66000

# Cache Configuration
cache:
  enabled: true
  provider: redis
  redis:
    host: localhost
    port: 6379
    password: ${REDIS_PASSWORD}
    db: 0
  ttl: 300
  maxSize: 1000

# Logging Configuration
logging:
  level: info
  format: json
  destination: stdout
  redact: ['password', 'token', 'secret']

# Security Configuration
security:
  cors:
    enabled: true
    origins: ['*']
    credentials: true
  helmet:
    enabled: true
  rateLimit:
    enabled: true
    max: 100
    window: 60000
  jwt:
    secret: ${JWT_SECRET}
    expiresIn: 3600

# Monitoring Configuration
monitoring:
  enabled: true
  metrics:
    port: 9090
    path: /metrics
  tracing:
    enabled: true
    endpoint: http://localhost:4318
    serviceName: fjas-api
```

### Environment Variables

```bash
# Server
FJAS_PORT=3000
FJAS_HOST=0.0.0.0
NODE_ENV=production

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=secret

# Security
JWT_SECRET=your-secret-key
API_KEY=your-api-key

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/db

# Monitoring
TELEMETRY_ENDPOINT=http://localhost:4318
```

### Programmatic Configuration

```javascript
const Fjas = require('fjas');

const app = new Fjas({
  server: {
    port: process.env.PORT || 3000,
    workers: require('os').cpus().length
  },
  cache: {
    enabled: true,
    provider: 'redis',
    redis: {
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT)
    }
  },
  logging: {
    level: process.env.LOG_LEVEL || 'info'
  }
});
```

## Error Handling

### Error Types

```typescript
class FjasError extends Error {
  statusCode: number;
  code: string;
  details?: any;
}

class ValidationError extends FjasError {
  statusCode = 400;
  code = 'VALIDATION_ERROR';
}

class NotFoundError extends FjasError {
  statusCode = 404;
  code = 'NOT_FOUND';
}

class UnauthorizedError extends FjasError {
  statusCode = 401;
  code = 'UNAUTHORIZED';
}
```

### Usage

```javascript
const { NotFoundError, ValidationError } = require('fjas/errors');

router.get('/users/:id', {
  handler: async (req, res) => {
    const user = await User.findById(req.params.id);
    
    if (!user) {
      throw new NotFoundError('User not found');
    }
    
    return res.json(user);
  }
});

// Global error handler
app.use(async (req, res, next) => {
  try {
    await next();
  } catch (error) {
    if (error instanceof FjasError) {
      return res.status(error.statusCode).json({
        error: error.message,
        code: error.code,
        details: error.details
      });
    }
    
    // Unexpected error
    console.error(error);
    return res.status(500).json({
      error: 'Internal server error'
    });
  }
});
```

## Next Steps

- [Configuration Guide](configuration.md) - Detailed configuration options
- [Performance Tuning](performance.md) - Optimize your application
- [Deployment Guide](deployment.md) - Deploy to production

---

[← Back to Documentation](../README.md#documentation)

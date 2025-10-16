# Getting Started with Fjas

Welcome to Fjas! This guide will help you get up and running in minutes.

## Prerequisites

Before you begin, ensure you have the following installed:

- **Node.js** 18.x or higher ([Download](https://nodejs.org/))
- **npm** or **yarn** package manager
- **Redis** 6.0+ (optional, for caching)
- **Git** for version control

## Installation Methods

### Method 1: NPM (Recommended)

```bash
# Install globally
npm install -g fjas

# Verify installation
fjas --version
```

### Method 2: Docker

```bash
# Pull the latest image
docker pull fjas/fjas:latest

# Run container
docker run -p 3000:3000 fjas/fjas:latest
```

### Method 3: From Source

```bash
# Clone repository
git clone https://github.com/vismathomas/fjas.git
cd fjas

# Install dependencies
npm install

# Build
npm run build

# Link globally
npm link
```

## Your First Fjas Application

Let's create a simple API service in just a few steps!

### Step 1: Initialize Project

```bash
# Create new directory
mkdir my-fjas-app
cd my-fjas-app

# Initialize Fjas project
fjas init
```

This creates the following structure:

```
my-fjas-app/
├── fjas.config.yml
├── routes/
│   └── index.js
├── middleware/
│   └── auth.js
└── package.json
```

### Step 2: Define Your First Route

Create `routes/users.js`:

```javascript
const { Router } = require('fjas');

const router = new Router();

// GET /users
router.get('/users', {
  description: 'Get all users',
  schema: {
    response: {
      200: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            id: { type: 'number' },
            name: { type: 'string' },
            email: { type: 'string', format: 'email' }
          }
        }
      }
    }
  },
  handler: async (req, res) => {
    const users = [
      { id: 1, name: 'Alice', email: 'alice@example.com' },
      { id: 2, name: 'Bob', email: 'bob@example.com' }
    ];
    return res.json(users);
  }
});

// GET /users/:id
router.get('/users/:id', {
  description: 'Get user by ID',
  schema: {
    params: {
      type: 'object',
      properties: {
        id: { type: 'number' }
      },
      required: ['id']
    }
  },
  handler: async (req, res) => {
    const userId = parseInt(req.params.id);
    const user = { id: userId, name: 'Alice', email: 'alice@example.com' };
    return res.json(user);
  }
});

module.exports = router;
```

### Step 3: Configure Your Application

Edit `fjas.config.yml`:

```yaml
server:
  host: 0.0.0.0
  port: 3000
  
logging:
  level: info
  format: json

cache:
  enabled: true
  provider: memory
  ttl: 300

security:
  cors:
    enabled: true
    origins: ['http://localhost:*']
  helmet:
    enabled: true
```

### Step 4: Start the Server

```bash
# Development mode with hot reload
fjas dev

# Production mode
fjas start
```

You should see:

```
🚀 Fjas server started
📍 Listening on http://localhost:3000
⚡ Ready to handle requests
```

### Step 5: Test Your API

```bash
# Using curl
curl http://localhost:3000/users

# Using httpie
http GET localhost:3000/users

# Using JavaScript fetch
fetch('http://localhost:3000/users')
  .then(res => res.json())
  .then(data => console.log(data));
```

## Working with Middleware

Middleware functions are executed before route handlers. Here's how to add authentication:

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = async (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};
```

Apply middleware to routes:

```javascript
const auth = require('./middleware/auth');

router.get('/protected', {
  middleware: [auth],
  handler: async (req, res) => {
    return res.json({ message: 'Protected data', user: req.user });
  }
});
```

## Request Lifecycle

Understanding the request flow in Fjas:

```mermaid
sequenceDiagram
    participant Client
    participant Fjas
    participant Middleware
    participant Cache
    participant Handler
    participant Database

    Client->>Fjas: HTTP Request
    Fjas->>Middleware: Process middleware chain
    Middleware->>Middleware: Authentication
    Middleware->>Middleware: Validation
    Middleware->>Cache: Check cache
    
    alt Cache Hit
        Cache-->>Client: Cached Response
    else Cache Miss
        Middleware->>Handler: Execute route handler
        Handler->>Database: Query data
        Database-->>Handler: Return data
        Handler->>Cache: Store in cache
        Handler-->>Client: JSON Response
    end
```

## Environment Variables

Create a `.env` file:

```bash
# Server Configuration
FJAS_PORT=3000
FJAS_HOST=0.0.0.0
NODE_ENV=development

# Redis Cache
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_password

# JWT Secret
JWT_SECRET=your_super_secret_key

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb

# Monitoring
TELEMETRY_ENDPOINT=http://localhost:4318
```

## CLI Commands

Fjas provides powerful CLI tools:

```bash
# Initialize new project
fjas init [project-name]

# Start server
fjas start

# Development mode with hot reload
fjas dev

# Generate route boilerplate
fjas generate route users

# Generate middleware
fjas generate middleware auth

# Run tests
fjas test

# Build for production
fjas build

# Display version
fjas --version

# Show help
fjas --help
```

## Next Steps

Congratulations! You've created your first Fjas application. Here's what to explore next:

- 📖 [Architecture Overview](architecture.md) - Understand how Fjas works
- 🔧 [Configuration Guide](configuration.md) - Advanced configuration options
- 🚀 [API Reference](api-reference.md) - Complete API documentation
- 📊 [Performance Tuning](performance.md) - Optimize your application
- 🐳 [Deployment Guide](deployment.md) - Deploy to production

## Troubleshooting

### Port Already in Use

```bash
# Find process using port 3000
netstat -ano | findstr :3000  # Windows
lsof -i :3000                  # macOS/Linux

# Kill the process or use different port
export FJAS_PORT=3001
fjas start
```

### Redis Connection Error

```bash
# Check Redis is running
redis-cli ping

# Start Redis (if not running)
redis-server

# Or disable cache temporarily
fjas start --no-cache
```

### Module Not Found

```bash
# Clear npm cache
npm cache clean --force

# Reinstall dependencies
rm -rf node_modules package-lock.json
npm install
```

## Getting Help

- 💬 [Discord Community](https://discord.gg/fjas)
- 📚 [Documentation](https://docs.fjas.io)
- 🐛 [GitHub Issues](https://github.com/vismathomas/fjas/issues)
- 📧 [Email Support](mailto:support@fjas.io)

---

**Ready for more?** Check out the [Architecture Overview](architecture.md) to understand Fjas internals.

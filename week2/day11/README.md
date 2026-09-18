# Day 11: REST API Best Practices

## 🎯 Learning Objectives

- Master REST API design principles and best practices
- Implement API versioning, caching, and rate limiting
- Build comprehensive error handling and status codes
- Create production-ready APIs with security and performance
- Implement API testing and documentation

## 📚 Theory & Concepts

### REST API Design
- **Resource-Based URLs**: Nouns, not verbs in URLs
- **HTTP Methods**: GET, POST, PUT, PATCH, DELETE semantics
- **Status Codes**: Proper HTTP status code usage
- **Content Negotiation**: Accept headers and response formats
- **Pagination**: Offset, limit, and cursor-based pagination

### API Best Practices
- **Versioning**: URL versioning, header versioning, content negotiation
- **Caching**: HTTP caching, ETags, cache-control headers
- **Rate Limiting**: Request throttling and quota management
- **Security**: Authentication, authorization, input validation
- **Documentation**: OpenAPI/Swagger specifications

### Performance Optimization
- **Response Compression**: Gzip compression for API responses
- **Database Optimization**: Query optimization and indexing
- **Caching Strategies**: Redis, in-memory, and HTTP caching
- **Connection Pooling**: Database connection management
- **Load Balancing**: Horizontal scaling and distribution

## 🛠️ Hands-on Tasks

### Task 1: Create REST API with Best Practices
Build a comprehensive REST API with proper design patterns:

```javascript
// routes/api/v1/index.js
const express = require('express');
const router = express.Router();

// Import route modules
const userRoutes = require('./userRoutes');
const productRoutes = require('./productRoutes');
const orderRoutes = require('./orderRoutes');
const analyticsRoutes = require('./analyticsRoutes');

// API versioning middleware
const apiVersion = (req, res, next) => {
  req.apiVersion = 'v1';
  next();
};

// Content negotiation middleware
const contentNegotiation = (req, res, next) => {
  const accept = req.headers.accept || 'application/json';
  
  if (accept.includes('application/json')) {
    req.responseFormat = 'json';
  } else if (accept.includes('application/xml')) {
    req.responseFormat = 'xml';
  } else {
    req.responseFormat = 'json'; // default
  }
  
  next();
};

// API documentation endpoint
router.get('/', (req, res) => {
  res.json({
    name: 'SDA Training API',
    version: '1.0.0',
    description: 'Advanced backend API for SDA training program',
    endpoints: {
      users: '/api/v1/users',
      products: '/api/v1/products',
      orders: '/api/v1/orders',
      analytics: '/api/v1/analytics'
    },
    documentation: '/api/v1/docs',
    status: 'operational'
  });
});

// Apply middleware
router.use(apiVersion);
router.use(contentNegotiation);

// Mount routes
router.use('/users', userRoutes);
router.use('/products', productRoutes);
router.use('/orders', orderRoutes);
router.use('/analytics', analyticsRoutes);

module.exports = router;
```

### Task 2: Implement API Versioning
Create comprehensive API versioning system:

```javascript
// middleware/apiVersioning.js
const express = require('express');
const { AppError } = require('./errorHandler');

const supportedVersions = ['v1', 'v2'];
const defaultVersion = 'v1';

const apiVersioning = (req, res, next) => {
  // Extract version from URL path
  const versionMatch = req.path.match(/^\/api\/(v\d+)/);
  const urlVersion = versionMatch ? versionMatch[1] : null;
  
  // Extract version from Accept header
  const acceptHeader = req.headers.accept || '';
  const headerVersion = acceptHeader.includes('version=') 
    ? acceptHeader.split('version=')[1].split(',')[0]
    : null;
  
  // Determine API version
  const apiVersion = urlVersion || headerVersion || defaultVersion;
  
  // Validate version
  if (!supportedVersions.includes(apiVersion)) {
    return next(new AppError(
      `API version ${apiVersion} is not supported. Supported versions: ${supportedVersions.join(', ')}`,
      400
    ));
  }
  
  req.apiVersion = apiVersion;
  next();
};

const versionSpecificRoutes = (version) => {
  return (req, res, next) => {
    req.versionSpecificRoutes = version;
    next();
  };
};

module.exports = { apiVersioning, versionSpecificRoutes };
```

### Task 3: Implement Caching Strategy
Create comprehensive caching system with Redis:

```javascript
// middleware/caching.js
const redis = require('redis');
const { logger } = require('./errorHandler');

class CacheService {
  constructor() {
    this.client = null;
    this.isConnected = false;
  }

  async connect() {
    try {
      this.client = redis.createClient({
        url: process.env.REDIS_URL || 'redis://localhost:6379',
        retry_strategy: (options) => {
          if (options.error && options.error.code === 'ECONNREFUSED') {
            return new Error('Redis server connection refused');
          }
          if (options.total_retry_time > 1000 * 60 * 60) {
            return new Error('Retry time exhausted');
          }
          if (options.attempt > 10) {
            return undefined;
          }
          return Math.min(options.attempt * 100, 3000);
        }
      });

      this.client.on('connect', () => {
        this.isConnected = true;
        logger.info('Redis connected successfully');
      });

      this.client.on('error', (err) => {
        this.isConnected = false;
        logger.error('Redis connection error:', err);
      });

      await this.client.connect();
    } catch (error) {
      logger.error('Redis connection failed:', error);
      throw error;
    }
  }

  async get(key) {
    try {
      if (!this.isConnected) return null;
      
      const value = await this.client.get(key);
      return value ? JSON.parse(value) : null;
    } catch (error) {
      logger.error('Redis get error:', error);
      return null;
    }
  }

  async set(key, value, ttl = 3600) {
    try {
      if (!this.isConnected) return false;
      
      await this.client.setEx(key, ttl, JSON.stringify(value));
      return true;
    } catch (error) {
      logger.error('Redis set error:', error);
      return false;
    }
  }

  async del(key) {
    try {
      if (!this.isConnected) return false;
      
      await this.client.del(key);
      return true;
    } catch (error) {
      logger.error('Redis delete error:', error);
      return false;
    }
  }

  async flush() {
    try {
      if (!this.isConnected) return false;
      
      await this.client.flushAll();
      return true;
    } catch (error) {
      logger.error('Redis flush error:', error);
      return false;
    }
  }

  generateKey(prefix, params = {}) {
    const sortedParams = Object.keys(params)
      .sort()
      .map(key => `${key}:${params[key]}`)
      .join('|');
    
    return `${prefix}:${sortedParams}`;
  }
}

const cacheService = new CacheService();

// Cache middleware
const cache = (ttl = 3600, keyGenerator = null) => {
  return async (req, res, next) => {
    try {
      // Generate cache key
      const cacheKey = keyGenerator 
        ? keyGenerator(req)
        : cacheService.generateKey(req.path, req.query);
      
      // Try to get from cache
      const cachedData = await cacheService.get(cacheKey);
      
      if (cachedData) {
        res.set('X-Cache', 'HIT');
        return res.json(cachedData);
      }
      
      // Store original json method
      const originalJson = res.json;
      
      // Override json method to cache response
      res.json = function(data) {
        // Cache the response
        cacheService.set(cacheKey, data, ttl);
        res.set('X-Cache', 'MISS');
        return originalJson.call(this, data);
      };
      
      next();
    } catch (error) {
      logger.error('Cache middleware error:', error);
      next();
    }
  };
};

module.exports = { cacheService, cache };
```

### Task 4: Implement Rate Limiting
Create comprehensive rate limiting system:

```javascript
// middleware/rateLimiting.js
const rateLimit = require('express-rate-limit');
const { AppError } = require('./errorHandler');

// General rate limiter
const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: {
    error: 'Too many requests from this IP, please try again later.',
    retryAfter: '15 minutes'
  },
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    res.status(429).json({
      success: false,
      error: {
        message: 'Too many requests from this IP, please try again later.',
        retryAfter: '15 minutes',
        limit: 100,
        remaining: 0
      }
    });
  }
});

// Strict rate limiter for sensitive endpoints
const strictLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // limit each IP to 5 requests per windowMs
  message: {
    error: 'Too many requests to sensitive endpoint, please try again later.',
    retryAfter: '15 minutes'
  },
  standardHeaders: true,
  legacyHeaders: false
});

// Login rate limiter
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // limit each IP to 5 login attempts per windowMs
  message: {
    error: 'Too many login attempts, please try again later.',
    retryAfter: '15 minutes'
  },
  standardHeaders: true,
  legacyHeaders: false,
  skipSuccessfulRequests: true // don't count successful requests
});

// API key rate limiter
const apiKeyLimiter = (req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  
  if (!apiKey) {
    return next();
  }
  
  // Different limits for different API keys
  const limits = {
    'free': { max: 100, windowMs: 15 * 60 * 1000 },
    'premium': { max: 1000, windowMs: 15 * 60 * 1000 },
    'enterprise': { max: 10000, windowMs: 15 * 60 * 1000 }
  };
  
  const limit = limits[apiKey] || limits['free'];
  
  const limiter = rateLimit({
    windowMs: limit.windowMs,
    max: limit.max,
    message: {
      error: 'API key rate limit exceeded',
      retryAfter: `${limit.windowMs / 1000 / 60} minutes`
    },
    standardHeaders: true,
    legacyHeaders: false
  });
  
  limiter(req, res, next);
};

module.exports = {
  generalLimiter,
  strictLimiter,
  loginLimiter,
  apiKeyLimiter
};
```

### Task 5: Create API Documentation
Implement comprehensive API documentation with Swagger:

```javascript
// docs/swagger.js
const swaggerJSDoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'SDA Training API',
      version: '1.0.0',
      description: 'Advanced backend API for SDA training program',
      contact: {
        name: 'API Support',
        email: 'support@sda-training.com'
      },
      license: {
        name: 'MIT',
        url: 'https://opensource.org/licenses/MIT'
      }
    },
    servers: [
      {
        url: 'http://localhost:3000/api/v1',
        description: 'Development server'
      },
      {
        url: 'https://api.sda-training.com/v1',
        description: 'Production server'
      }
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT'
        },
        apiKey: {
          type: 'apiKey',
          in: 'header',
          name: 'X-API-Key'
        }
      },
      schemas: {
        User: {
          type: 'object',
          required: ['name', 'email', 'password'],
          properties: {
            id: {
              type: 'string',
              format: 'uuid',
              description: 'User unique identifier'
            },
            name: {
              type: 'string',
              minLength: 2,
              maxLength: 50,
              description: 'User full name'
            },
            email: {
              type: 'string',
              format: 'email',
              description: 'User email address'
            },
            role: {
              type: 'string',
              enum: ['user', 'admin', 'moderator'],
              description: 'User role'
            },
            isActive: {
              type: 'boolean',
              description: 'User account status'
            },
            createdAt: {
              type: 'string',
              format: 'date-time',
              description: 'User creation timestamp'
            },
            updatedAt: {
              type: 'string',
              format: 'date-time',
              description: 'User last update timestamp'
            }
          }
        },
        Product: {
          type: 'object',
          required: ['name', 'description', 'price', 'category'],
          properties: {
            id: {
              type: 'string',
              format: 'uuid',
              description: 'Product unique identifier'
            },
            name: {
              type: 'string',
              minLength: 2,
              maxLength: 100,
              description: 'Product name'
            },
            description: {
              type: 'string',
              minLength: 10,
              maxLength: 500,
              description: 'Product description'
            },
            price: {
              type: 'number',
              minimum: 0,
              description: 'Product price'
            },
            category: {
              type: 'string',
              description: 'Product category'
            },
            stock: {
              type: 'integer',
              minimum: 0,
              description: 'Product stock quantity'
            },
            imageUrl: {
              type: 'string',
              format: 'uri',
              description: 'Product image URL'
            },
            tags: {
              type: 'array',
              items: {
                type: 'string'
              },
              description: 'Product tags'
            }
          }
        },
        Error: {
          type: 'object',
          properties: {
            success: {
              type: 'boolean',
              example: false
            },
            error: {
              type: 'object',
              properties: {
                message: {
                  type: 'string',
                  description: 'Error message'
                },
                code: {
                  type: 'string',
                  description: 'Error code'
                },
                details: {
                  type: 'array',
                  items: {
                    type: 'object'
                  },
                  description: 'Error details'
                }
              }
            }
          }
        }
      }
    },
    security: [
      {
        bearerAuth: []
      },
      {
        apiKey: []
      }
    ]
  },
  apis: ['./routes/**/*.js', './models/**/*.js']
};

const specs = swaggerJSDoc(options);

module.exports = { specs, swaggerUi };
```

## 📝 Documentation Tasks

### Create API Design Guide
Create `week2/day11/docs/api-design.md`:

```markdown
# REST API Design Guide

## API Design Principles
- **Resource-Based URLs**: Use nouns, not verbs
- **HTTP Methods**: Proper use of GET, POST, PUT, PATCH, DELETE
- **Status Codes**: Consistent HTTP status code usage
- **Content Negotiation**: Accept headers and response formats
- **Pagination**: Offset, limit, and cursor-based pagination

## Best Practices
- **Versioning**: URL versioning and header versioning
- **Caching**: HTTP caching and ETags
- **Rate Limiting**: Request throttling and quota management
- **Security**: Authentication, authorization, input validation
- **Documentation**: OpenAPI/Swagger specifications

## Performance Optimization
- **Response Compression**: Gzip compression
- **Database Optimization**: Query optimization and indexing
- **Caching Strategies**: Redis, in-memory, and HTTP caching
- **Connection Pooling**: Database connection management
- **Load Balancing**: Horizontal scaling and distribution
```

## 🧪 Testing & Validation

### API Testing
- [x] All endpoints work correctly
- [x] Error handling works
- [x] Rate limiting works
- [x] Caching works
- [x] Documentation is accurate

### Performance Testing
- [x] Response times are acceptable
- [x] Rate limiting works correctly
- [x] Caching improves performance
- [x] Database queries are optimized
- [x] Memory usage is acceptable

## 📊 Success Criteria

By the end of Day 11, you should have:

✅ **REST API Mastery**: Production-ready API design  
✅ **Versioning**: Comprehensive API versioning  
✅ **Caching**: Redis caching implementation  
✅ **Rate Limiting**: Request throttling and quota management  
✅ **Documentation**: Complete API documentation  

## ✅ Completion Status

All Day 11 deliverables are complete and tested:
- `server/routes/api/v1/` — versioned user/product/order/analytics/notification/health routes
- `server/middleware/` — apiVersioning, caching, rateLimiting (+ auth, errorHandler, logger, performance, validation)
- `server/database/` + `server/migrations/` + `server/services/`
- `docs/api-design.md`, `docs/swagger.js`

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 11: REST API Best Practices"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 12**: Review authentication and authorization
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [REST API Design](https://restfulapi.net/)
- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Rate Limiting](https://expressjs.com/en/advanced/best-practice-performance.html)

---

**Ready for Day 12? Check out [Day 12: Authentication & RBAC](../day12/README.md)!** 🚀

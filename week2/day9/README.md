# Day 9: Express & Middleware

## 🎯 Learning Objectives

- Master Express.js framework and middleware architecture
- Implement custom middleware for authentication, validation, and logging
- Build robust error handling and validation systems
- Create modular route handlers with proper separation of concerns
- Optimize Express applications for production use

## 📚 Theory & Concepts

### Express.js Fundamentals
- **Middleware**: Request/response cycle, order of execution
- **Routing**: Route parameters, query strings, request body
- **Templates**: View engines, template rendering
- **Static Files**: Serving static assets, CDN integration
- **Security**: CORS, helmet, rate limiting, input validation

### Middleware Architecture
- **Application Middleware**: Global middleware for all routes
- **Router Middleware**: Route-specific middleware
- **Error Middleware**: Error handling and recovery
- **Third-party Middleware**: External libraries and tools
- **Custom Middleware**: Application-specific logic

### Best Practices
- **Separation of Concerns**: Controller, service, and data layers
- **Error Handling**: Centralized error management
- **Validation**: Input validation and sanitization
- **Security**: Authentication, authorization, and protection
- **Performance**: Caching, compression, and optimization

## 🛠️ Hands-on Tasks

### Task 1: Create Express Application with Middleware
Build a comprehensive Express application with custom middleware:

```javascript
// server/app.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');
const rateLimit = require('express-rate-limit');
const morgan = require('morgan');
const { body, validationResult } = require('express-validator');

// Import custom middleware
const authMiddleware = require('./middleware/auth');
const validationMiddleware = require('./middleware/validation');
const errorHandler = require('./middleware/errorHandler');
const logger = require('./middleware/logger');
const performanceMiddleware = require('./middleware/performance');

// Import routes
const userRoutes = require('./routes/userRoutes');
const productRoutes = require('./routes/productRoutes');
const orderRoutes = require('./routes/orderRoutes');

class ExpressApp {
  constructor() {
    this.app = express();
    this.port = process.env.PORT || 3000;
    this.setupMiddleware();
    this.setupRoutes();
    this.setupErrorHandling();
  }

  setupMiddleware() {
    // Security middleware
    this.app.use(helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          styleSrc: ["'self'", "'unsafe-inline'"],
          scriptSrc: ["'self'"],
          imgSrc: ["'self'", "data:", "https:"],
        },
      },
    }));

    // CORS configuration
    this.app.use(cors({
      origin: process.env.FRONTEND_URL || 'http://localhost:3000',
      credentials: true,
      methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
      allowedHeaders: ['Content-Type', 'Authorization']
    }));

    // Compression
    this.app.use(compression());

    // Rate limiting
    const limiter = rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // limit each IP to 100 requests per windowMs
      message: {
        error: 'Too many requests from this IP, please try again later.'
      },
      standardHeaders: true,
      legacyHeaders: false,
    });
    this.app.use('/api/', limiter);

    // Logging
    this.app.use(morgan('combined'));
    this.app.use(logger);

    // Performance monitoring
    this.app.use(performanceMiddleware);

    // Body parsing
    this.app.use(express.json({ limit: '10mb' }));
    this.app.use(express.urlencoded({ extended: true, limit: '10mb' }));

    // Request validation
    this.app.use(validationMiddleware);
  }

  setupRoutes() {
    // Health check
    this.app.get('/health', (req, res) => {
      res.json({
        status: 'OK',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
        memory: process.memoryUsage(),
        version: process.version
      });
    });

    // API routes
    this.app.use('/api/users', userRoutes);
    this.app.use('/api/products', productRoutes);
    this.app.use('/api/orders', orderRoutes);

    // 404 handler
    this.app.use('*', (req, res) => {
      res.status(404).json({
        success: false,
        message: 'Route not found',
        path: req.originalUrl
      });
    });
  }

  setupErrorHandling() {
    this.app.use(errorHandler);
  }

  start() {
    this.app.listen(this.port, () => {
      console.log(`Server running on port ${this.port}`);
      console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
    });
  }
}

module.exports = ExpressApp;
```

### Task 2: Implement Custom Middleware
Create comprehensive middleware for authentication, validation, and logging:

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const { AppError } = require('./errorHandler');

const authMiddleware = (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new AppError('Access token is required', 401);
    }

    const token = authHeader.substring(7);
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    req.user = decoded;
    next();
  } catch (error) {
    if (error.name === 'JsonWebTokenError') {
      next(new AppError('Invalid token', 401));
    } else if (error.name === 'TokenExpiredError') {
      next(new AppError('Token expired', 401));
    } else {
      next(error);
    }
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!req.user) {
      return next(new AppError('Authentication required', 401));
    }

    if (!roles.includes(req.user.role)) {
      return next(new AppError('Insufficient permissions', 403));
    }

    next();
  };
};

const optionalAuth = (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (authHeader && authHeader.startsWith('Bearer ')) {
      const token = authHeader.substring(7);
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req.user = decoded;
    }
    
    next();
  } catch (error) {
    // Continue without authentication
    next();
  }
};

module.exports = { authMiddleware, authorize, optionalAuth };
```

### Task 3: Create Validation Middleware
Implement comprehensive input validation:

```javascript
// middleware/validation.js
const { body, param, query, validationResult } = require('express-validator');
const { AppError } = require('./errorHandler');

const handleValidationErrors = (req, res, next) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    const errorMessages = errors.array().map(error => ({
      field: error.path,
      message: error.msg,
      value: error.value
    }));
    
    return next(new AppError('Validation failed', 400, errorMessages));
  }
  next();
};

// User validation rules
const validateUser = [
  body('name')
    .trim()
    .isLength({ min: 2, max: 50 })
    .withMessage('Name must be between 2 and 50 characters'),
  body('email')
    .isEmail()
    .normalizeEmail()
    .withMessage('Valid email is required'),
  body('password')
    .isLength({ min: 8 })
    .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/)
    .withMessage('Password must be at least 8 characters with uppercase, lowercase, number and special character'),
  body('role')
    .optional()
    .isIn(['user', 'admin', 'moderator'])
    .withMessage('Role must be user, admin, or moderator'),
  handleValidationErrors
];

const validateLogin = [
  body('email')
    .isEmail()
    .normalizeEmail()
    .withMessage('Valid email is required'),
  body('password')
    .notEmpty()
    .withMessage('Password is required'),
  handleValidationErrors
];

const validateProduct = [
  body('name')
    .trim()
    .isLength({ min: 2, max: 100 })
    .withMessage('Product name must be between 2 and 100 characters'),
  body('description')
    .trim()
    .isLength({ min: 10, max: 500 })
    .withMessage('Description must be between 10 and 500 characters'),
  body('price')
    .isFloat({ min: 0 })
    .withMessage('Price must be a positive number'),
  body('category')
    .trim()
    .isLength({ min: 2, max: 50 })
    .withMessage('Category must be between 2 and 50 characters'),
  body('stock')
    .isInt({ min: 0 })
    .withMessage('Stock must be a non-negative integer'),
  handleValidationErrors
];

const validateOrder = [
  body('items')
    .isArray({ min: 1 })
    .withMessage('Order must contain at least one item'),
  body('items.*.productId')
    .isUUID()
    .withMessage('Product ID must be a valid UUID'),
  body('items.*.quantity')
    .isInt({ min: 1 })
    .withMessage('Quantity must be a positive integer'),
  body('shippingAddress')
    .isObject()
    .withMessage('Shipping address is required'),
  body('shippingAddress.street')
    .trim()
    .isLength({ min: 5, max: 100 })
    .withMessage('Street address must be between 5 and 100 characters'),
  body('shippingAddress.city')
    .trim()
    .isLength({ min: 2, max: 50 })
    .withMessage('City must be between 2 and 50 characters'),
  body('shippingAddress.zipCode')
    .isPostalCode('US')
    .withMessage('Valid ZIP code is required'),
  handleValidationErrors
];

const validateId = [
  param('id')
    .isUUID()
    .withMessage('Invalid ID format'),
  handleValidationErrors
];

const validatePagination = [
  query('page')
    .optional()
    .isInt({ min: 1 })
    .withMessage('Page must be a positive integer'),
  query('limit')
    .optional()
    .isInt({ min: 1, max: 100 })
    .withMessage('Limit must be between 1 and 100'),
  query('sort')
    .optional()
    .isIn(['createdAt', 'updatedAt', 'name', 'price'])
    .withMessage('Invalid sort field'),
  query('order')
    .optional()
    .isIn(['asc', 'desc'])
    .withMessage('Order must be asc or desc'),
  handleValidationErrors
];

module.exports = {
  validateUser,
  validateLogin,
  validateProduct,
  validateOrder,
  validateId,
  validatePagination,
  handleValidationErrors
};
```

### Task 4: Create Route Handlers
Implement comprehensive route handlers with proper error handling:

```javascript
// routes/userRoutes.js
const express = require('express');
const router = express.Router();
const userService = require('../services/userService');
const { authMiddleware, authorize } = require('../middleware/auth');
const { validateUser, validateLogin, validateId, validatePagination } = require('../middleware/validation');

// Public routes
router.post('/register', validateUser, async (req, res, next) => {
  try {
    const user = await userService.createUser(req.body);
    res.status(201).json({
      success: true,
      message: 'User created successfully',
      data: user
    });
  } catch (error) {
    next(error);
  }
});

router.post('/login', validateLogin, async (req, res, next) => {
  try {
    const { email, password } = req.body;
    const result = await userService.authenticateUser(email, password);
    
    res.json({
      success: true,
      message: 'Login successful',
      data: result
    });
  } catch (error) {
    next(error);
  }
});

// Protected routes
router.use(authMiddleware);

router.get('/profile', async (req, res, next) => {
  try {
    const user = await userService.getUserById(req.user.userId);
    res.json({
      success: true,
      data: user
    });
  } catch (error) {
    next(error);
  }
});

router.put('/profile', validateUser, async (req, res, next) => {
  try {
    const user = await userService.updateUser(req.user.userId, req.body);
    res.json({
      success: true,
      message: 'Profile updated successfully',
      data: user
    });
  } catch (error) {
    next(error);
  }
});

router.delete('/profile', async (req, res, next) => {
  try {
    await userService.deleteUser(req.user.userId);
    res.json({
      success: true,
      message: 'Account deleted successfully'
    });
  } catch (error) {
    next(error);
  }
});

router.post('/logout', async (req, res, next) => {
  try {
    await userService.logout(req.user.userId);
    res.json({
      success: true,
      message: 'Logged out successfully'
    });
  } catch (error) {
    next(error);
  }
});

// Admin routes
router.get('/', authorize('admin'), validatePagination, async (req, res, next) => {
  try {
    const { page, limit, sort, order, role, isActive } = req.query;
    const filters = { role, isActive };
    const options = { page, limit, sort, order };
    
    const result = await userService.getAllUsers(filters, options);
    res.json({
      success: true,
      data: result
    });
  } catch (error) {
    next(error);
  }
});

router.get('/:id', authorize('admin'), validateId, async (req, res, next) => {
  try {
    const user = await userService.getUserById(req.params.id);
    res.json({
      success: true,
      data: user
    });
  } catch (error) {
    next(error);
  }
});

router.put('/:id', authorize('admin'), validateId, validateUser, async (req, res, next) => {
  try {
    const user = await userService.updateUser(req.params.id, req.body);
    res.json({
      success: true,
      message: 'User updated successfully',
      data: user
    });
  } catch (error) {
    next(error);
  }
});

router.delete('/:id', authorize('admin'), validateId, async (req, res, next) => {
  try {
    await userService.deleteUser(req.params.id);
    res.json({
      success: true,
      message: 'User deleted successfully'
    });
  } catch (error) {
    next(error);
  }
});

module.exports = router;
```

### Task 5: Implement Error Handling
Create comprehensive error handling system:

```javascript
// middleware/errorHandler.js
const winston = require('winston');

// Configure logger
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'express-app' },
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

class AppError extends Error {
  constructor(message, statusCode, details = null) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
    this.details = details;
    Error.captureStackTrace(this, this.constructor);
  }
}

const errorHandler = (err, req, res, next) => {
  let error = { ...err };
  error.message = err.message;

  // Log error
  logger.error({
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    ip: req.ip,
    userAgent: req.get('User-Agent'),
    userId: req.user?.userId
  });

  // Mongoose bad ObjectId
  if (err.name === 'CastError') {
    const message = 'Resource not found';
    error = new AppError(message, 404);
  }

  // Mongoose duplicate key
  if (err.code === 11000) {
    const message = 'Duplicate field value entered';
    error = new AppError(message, 400);
  }

  // Mongoose validation error
  if (err.name === 'ValidationError') {
    const message = Object.values(err.errors).map(val => val.message);
    error = new AppError(message, 400);
  }

  // JWT errors
  if (err.name === 'JsonWebTokenError') {
    const message = 'Invalid token';
    error = new AppError(message, 401);
  }

  if (err.name === 'TokenExpiredError') {
    const message = 'Token expired';
    error = new AppError(message, 401);
  }

  // Rate limit error
  if (err.status === 429) {
    const message = 'Too many requests, please try again later';
    error = new AppError(message, 429);
  }

  res.status(error.statusCode || 500).json({
    success: false,
    error: {
      message: error.message || 'Server Error',
      ...(error.details && { details: error.details }),
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
    }
  });
};

module.exports = { errorHandler, AppError, logger };
```

## 📝 Documentation Tasks

### Create Express Architecture Guide
Create `week2/day9/docs/express-architecture.md`:

```markdown
# Express.js Architecture Guide

## Application Structure
- **Middleware Stack**: Request/response cycle with proper order
- **Route Handlers**: Modular route organization
- **Error Handling**: Centralized error management
- **Validation**: Input validation and sanitization
- **Security**: Authentication, authorization, and protection

## Middleware Best Practices
- **Order Matters**: Middleware execution order is critical
- **Error Handling**: Always include error middleware last
- **Security First**: Security middleware should be early in the stack
- **Performance**: Optimize middleware for production use
- **Logging**: Comprehensive logging for debugging and monitoring

## Route Organization
- **Modular Routes**: Separate route files for different resources
- **Middleware Integration**: Proper middleware usage in routes
- **Validation**: Input validation for all routes
- **Error Handling**: Consistent error responses
- **Documentation**: Clear route documentation
```

## 🧪 Testing & Validation

### Middleware Testing
- [x] Authentication middleware works correctly
- [x] Validation middleware catches errors
- [x] Error handling middleware works
- [x] Performance middleware tracks metrics
- [x] Logging middleware records requests

### Route Testing
- [x] All routes respond correctly
- [x] Error handling works
- [x] Validation works
- [x] Authentication works
- [x] Authorization works

## 📊 Success Criteria

By the end of Day 9, you should have:

✅ **Express Mastery**: Advanced Express.js development  
✅ **Middleware Architecture**: Custom middleware implementation  
✅ **Error Handling**: Comprehensive error management  
✅ **Validation**: Input validation and sanitization  
✅ **Security**: Authentication and authorization  

## ✅ Completion Status

All Day 9 deliverables are complete and tested:
- `server/index.js` — clustered Express + Socket.io app (middleware, routes, services, WS, error handling)
- `server/middleware/` — auth, validation (per-route chains), errorHandler, logger, performance (global timing, wired in `setupMiddleware`)
- `server/routes/` — health, user, product, order routes
- `server/services/` — user, product, order, notification services
- `docs/express-architecture.md`
- Note: stray root `app.js` (broken relative requires) removed; `server/index.js` is the entry point

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 9: Express & Middleware"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 10**: Review database concepts and MongoDB
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [Express.js Documentation](https://expressjs.com/)
- [Express Middleware](https://expressjs.com/en/guide/using-middleware.html)
- [Express Security](https://expressjs.com/en/advanced/best-practice-security.html)
- [Express Best Practices](https://expressjs.com/en/advanced/best-practice-performance.html)

---

**Ready for Day 10? Check out [Day 10: MongoDB & SQL](../day10/README.md)!** 🚀

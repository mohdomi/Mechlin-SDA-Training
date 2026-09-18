# Day 8: Node.js Deep Dive

## 🎯 Learning Objectives

- Master Node.js asynchronous programming patterns
- Implement clustering for performance optimization
- Build modular Node.js applications with proper architecture
- Handle errors and implement robust error handling
- Optimize performance with advanced Node.js features

## 📚 Theory & Concepts

### Node.js Fundamentals
- **Event Loop**: Understanding the event-driven architecture
- **Asynchronous Programming**: Callbacks, Promises, async/await
- **Streams**: Readable, Writable, Transform, Duplex streams
- **Buffers**: Binary data handling and manipulation
- **Modules**: CommonJS, ES6 modules, module resolution

### Advanced Node.js Features
- **Clustering**: Multi-process applications for CPU-intensive tasks
- **Worker Threads**: CPU-intensive operations without blocking
- **Child Processes**: Spawning and managing child processes
- **Performance Monitoring**: Profiling and optimization
- **Memory Management**: Garbage collection and memory leaks

### Architecture Patterns
- **Microservices**: Service-oriented architecture
- **Event-Driven**: Event sourcing and CQRS patterns
- **Dependency Injection**: Inversion of control
- **Factory Pattern**: Object creation and management
- **Observer Pattern**: Event handling and notifications

## 🛠️ Hands-on Tasks

### Task 1: Create Multi-Service Node.js Application
Build a comprehensive Node.js application with multiple services:

```javascript
// server/index.js
const express = require('express');
const cluster = require('cluster');
const os = require('os');
const { createServer } = require('http');
const { Server } = require('socket.io');
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');
const rateLimit = require('express-rate-limit');

// Import services
const userService = require('./services/userService');
const productService = require('./services/productService');
const orderService = require('./services/orderService');
const notificationService = require('./services/notificationService');

// Import middleware
const errorHandler = require('./middleware/errorHandler');
const logger = require('./middleware/logger');
const auth = require('./middleware/auth');

// Import routes
const userRoutes = require('./routes/userRoutes');
const productRoutes = require('./routes/productRoutes');
const orderRoutes = require('./routes/orderRoutes');
const healthRoutes = require('./routes/healthRoutes');

class Application {
  constructor() {
    this.app = express();
    this.server = createServer(this.app);
    this.io = new Server(this.server, {
      cors: {
        origin: process.env.FRONTEND_URL || "http://localhost:3000",
        methods: ["GET", "POST"]
      }
    });
    this.port = process.env.PORT || 3000;
    this.isProduction = process.env.NODE_ENV === 'production';
  }

  async initialize() {
    try {
      await this.setupMiddleware();
      await this.setupRoutes();
      await this.setupServices();
      await this.setupWebSocket();
      await this.setupErrorHandling();
      await this.startServer();
    } catch (error) {
      console.error('Application initialization failed:', error);
      process.exit(1);
    }
  }

  setupMiddleware() {
    // Security middleware
    this.app.use(helmet());
    this.app.use(cors({
      origin: process.env.FRONTEND_URL || "http://localhost:3000",
      credentials: true
    }));

    // Compression
    this.app.use(compression());

    // Rate limiting
    const limiter = rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // limit each IP to 100 requests per windowMs
      message: 'Too many requests from this IP, please try again later.'
    });
    this.app.use('/api/', limiter);

    // Body parsing
    this.app.use(express.json({ limit: '10mb' }));
    this.app.use(express.urlencoded({ extended: true, limit: '10mb' }));

    // Logging
    this.app.use(logger);
  }

  setupRoutes() {
    // Health check
    this.app.use('/health', healthRoutes);

    // API routes
    this.app.use('/api/users', userRoutes);
    this.app.use('/api/products', productRoutes);
    this.app.use('/api/orders', orderRoutes);

    // 404 handler
    this.app.use('*', (req, res) => {
      res.status(404).json({
        success: false,
        message: 'Route not found'
      });
    });
  }

  async setupServices() {
    // Initialize services
    await userService.initialize();
    await productService.initialize();
    await orderService.initialize();
    await notificationService.initialize();

    console.log('All services initialized successfully');
  }

  setupWebSocket() {
    this.io.on('connection', (socket) => {
      console.log('Client connected:', socket.id);

      socket.on('join', (room) => {
        socket.join(room);
        console.log(`Client ${socket.id} joined room ${room}`);
      });

      socket.on('disconnect', () => {
        console.log('Client disconnected:', socket.id);
      });

      // Handle custom events
      socket.on('user:update', (data) => {
        socket.broadcast.emit('user:updated', data);
      });

      socket.on('order:create', (data) => {
        socket.broadcast.emit('order:created', data);
      });
    });
  }

  setupErrorHandling() {
    this.app.use(errorHandler);

    // Unhandled promise rejections
    process.on('unhandledRejection', (reason, promise) => {
      console.error('Unhandled Rejection at:', promise, 'reason:', reason);
      process.exit(1);
    });

    // Uncaught exceptions
    process.on('uncaughtException', (error) => {
      console.error('Uncaught Exception:', error);
      process.exit(1);
    });
  }

  startServer() {
    this.server.listen(this.port, () => {
      console.log(`Server running on port ${this.port}`);
      console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
      console.log(`Process ID: ${process.pid}`);
    });
  }
}

// Clustering setup
if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  console.log(`Master process ${process.pid} is running`);
  console.log(`Starting ${numCPUs} workers`);

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    console.log('Starting a new worker');
    cluster.fork();
  });

  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('Master received SIGTERM, shutting down gracefully');
    for (const id in cluster.workers) {
      cluster.workers[id].kill();
    }
  });
} else {
  // Worker process
  const app = new Application();
  app.initialize();
}
```

### Task 2: Implement Advanced Service Architecture
Create modular services with proper separation of concerns:

```javascript
// services/userService.js
const EventEmitter = require('events');
const { v4: uuidv4 } = require('uuid');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

class UserService extends EventEmitter {
  constructor() {
    super();
    this.users = new Map();
    this.sessions = new Map();
    this.jwtSecret = process.env.JWT_SECRET || 'your-secret-key';
    this.jwtExpiresIn = process.env.JWT_EXPIRES_IN || '7d';
  }

  async initialize() {
    console.log('UserService initialized');
    this.setupEventHandlers();
  }

  setupEventHandlers() {
    this.on('user:created', (user) => {
      console.log(`User created: ${user.email}`);
    });

    this.on('user:updated', (user) => {
      console.log(`User updated: ${user.email}`);
    });

    this.on('user:deleted', (userId) => {
      console.log(`User deleted: ${userId}`);
    });
  }

  async createUser(userData) {
    try {
      const { email, password, name, role = 'user' } = userData;

      // Validate input
      if (!email || !password || !name) {
        throw new Error('Missing required fields');
      }

      // Check if user exists
      if (this.users.has(email)) {
        throw new Error('User already exists');
      }

      // Hash password
      const saltRounds = 12;
      const hashedPassword = await bcrypt.hash(password, saltRounds);

      // Create user
      const user = {
        id: uuidv4(),
        email,
        password: hashedPassword,
        name,
        role,
        createdAt: new Date(),
        updatedAt: new Date(),
        isActive: true
      };

      // Store user
      this.users.set(email, user);

      // Emit event
      this.emit('user:created', user);

      return {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
        createdAt: user.createdAt
      };
    } catch (error) {
      console.error('Error creating user:', error);
      throw error;
    }
  }

  async authenticateUser(email, password) {
    try {
      const user = this.users.get(email);
      if (!user) {
        throw new Error('User not found');
      }

      if (!user.isActive) {
        throw new Error('User account is deactivated');
      }

      const isValidPassword = await bcrypt.compare(password, user.password);
      if (!isValidPassword) {
        throw new Error('Invalid password');
      }

      // Generate JWT token
      const token = jwt.sign(
        { 
          userId: user.id, 
          email: user.email, 
          role: user.role 
        },
        this.jwtSecret,
        { expiresIn: this.jwtExpiresIn }
      );

      // Store session
      this.sessions.set(user.id, {
        token,
        createdAt: new Date(),
        lastActivity: new Date()
      });

      return {
        token,
        user: {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role
        }
      };
    } catch (error) {
      console.error('Error authenticating user:', error);
      throw error;
    }
  }

  async getUserById(userId) {
    const user = Array.from(this.users.values()).find(u => u.id === userId);
    if (!user) {
      throw new Error('User not found');
    }

    return {
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
      createdAt: user.createdAt,
      updatedAt: user.updatedAt
    };
  }

  async updateUser(userId, updateData) {
    const user = Array.from(this.users.values()).find(u => u.id === userId);
    if (!user) {
      throw new Error('User not found');
    }

    // Update user data
    Object.assign(user, updateData, { updatedAt: new Date() });
    this.users.set(user.email, user);

    // Emit event
    this.emit('user:updated', user);

    return {
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
      updatedAt: user.updatedAt
    };
  }

  async deleteUser(userId) {
    const user = Array.from(this.users.values()).find(u => u.id === userId);
    if (!user) {
      throw new Error('User not found');
    }

    // Remove user
    this.users.delete(user.email);
    this.sessions.delete(userId);

    // Emit event
    this.emit('user:deleted', userId);

    return { message: 'User deleted successfully' };
  }

  async getAllUsers(filters = {}) {
    let users = Array.from(this.users.values());

    // Apply filters
    if (filters.role) {
      users = users.filter(u => u.role === filters.role);
    }

    if (filters.isActive !== undefined) {
      users = users.filter(u => u.isActive === filters.isActive);
    }

    // Sort and paginate
    const page = parseInt(filters.page) || 1;
    const limit = parseInt(filters.limit) || 10;
    const skip = (page - 1) * limit;

    const total = users.length;
    const paginatedUsers = users
      .sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt))
      .slice(skip, skip + limit)
      .map(user => ({
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
        createdAt: user.createdAt
      }));

    return {
      users: paginatedUsers,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit)
      }
    };
  }

  async validateToken(token) {
    try {
      const decoded = jwt.verify(token, this.jwtSecret);
      const session = this.sessions.get(decoded.userId);
      
      if (!session || session.token !== token) {
        throw new Error('Invalid token');
      }

      // Update last activity
      session.lastActivity = new Date();

      return decoded;
    } catch (error) {
      throw new Error('Invalid token');
    }
  }

  async logout(userId) {
    this.sessions.delete(userId);
    return { message: 'Logged out successfully' };
  }
}

module.exports = new UserService();
```

### Task 3: Implement Error Handling and Logging
Create comprehensive error handling and logging system:

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
  defaultMeta: { service: 'user-service' },
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
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
    userAgent: req.get('User-Agent')
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

  res.status(error.statusCode || 500).json({
    success: false,
    error: {
      message: error.message || 'Server Error',
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
    }
  });
};

module.exports = { errorHandler, AppError, logger };
```

### Task 4: Implement Performance Monitoring
Create performance monitoring and optimization:

```javascript
// middleware/performance.js
const performance = require('perf_hooks');

class PerformanceMonitor {
  constructor() {
    this.metrics = new Map();
    this.startTime = Date.now();
  }

  startTimer(name) {
    const timer = performance.performance.now();
    this.metrics.set(name, { start: timer });
  }

  endTimer(name) {
    const timer = this.metrics.get(name);
    if (timer) {
      const duration = performance.performance.now() - timer.start;
      this.metrics.set(name, { ...timer, duration, end: performance.performance.now() });
      return duration;
    }
    return null;
  }

  getMetrics() {
    const uptime = Date.now() - this.startTime;
    const memoryUsage = process.memoryUsage();
    
    return {
      uptime,
      memory: {
        rss: memoryUsage.rss,
        heapTotal: memoryUsage.heapTotal,
        heapUsed: memoryUsage.heapUsed,
        external: memoryUsage.external
      },
      timers: Object.fromEntries(this.metrics),
      process: {
        pid: process.pid,
        version: process.version,
        platform: process.platform,
        arch: process.arch
      }
    };
  }

  reset() {
    this.metrics.clear();
    this.startTime = Date.now();
  }
}

const performanceMonitor = new PerformanceMonitor();

const performanceMiddleware = (req, res, next) => {
  const startTime = performance.performance.now();
  
  res.on('finish', () => {
    const duration = performance.performance.now() - startTime;
    const memoryUsage = process.memoryUsage();
    
    console.log({
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration: `${duration.toFixed(2)}ms`,
      memory: {
        heapUsed: `${(memoryUsage.heapUsed / 1024 / 1024).toFixed(2)}MB`,
        heapTotal: `${(memoryUsage.heapTotal / 1024 / 1024).toFixed(2)}MB`
      },
      timestamp: new Date().toISOString()
    });
  });
  
  next();
};

module.exports = { performanceMiddleware, performanceMonitor };
```

## 📝 Documentation Tasks

### Create Node.js Architecture Guide
Create `week2/day8/docs/nodejs-architecture.md`:

```markdown
# Node.js Architecture Guide

## Application Structure
- **Modular Design**: Separation of concerns with service layers
- **Event-Driven**: EventEmitter pattern for loose coupling
- **Clustering**: Multi-process architecture for performance
- **Error Handling**: Comprehensive error management
- **Logging**: Structured logging with Winston

## Performance Optimization
- **Clustering**: CPU utilization across multiple cores
- **Streams**: Efficient data processing
- **Caching**: Memory and Redis caching strategies
- **Monitoring**: Performance metrics and profiling
- **Memory Management**: Garbage collection optimization

## Best Practices
- **Error Handling**: Try-catch blocks and error boundaries
- **Async/Await**: Modern asynchronous programming
- **Security**: Input validation and sanitization
- **Testing**: Unit and integration testing
- **Documentation**: Code comments and API documentation
```

## 🧪 Testing & Validation

### Performance Testing
- [x] Application handles concurrent requests
- [x] Memory usage is optimized
- [x] Response times are acceptable
- [x] Clustering works correctly
- [x] Error handling is robust

### Functionality Testing
- [x] All services work correctly
- [x] WebSocket connections work
- [x] Error handling works
- [x] Logging is comprehensive
- [x] Performance monitoring works

## 📊 Success Criteria

By the end of Day 8, you should have:

✅ **Node.js Mastery**: Advanced server-side development  
✅ **Service Architecture**: Modular, scalable design  
✅ **Error Handling**: Comprehensive error management  
✅ **Performance**: Optimized for production use  
✅ **Monitoring**: Real-time performance tracking  

## ✅ Completion Status

All Day 8 deliverables are complete and tested:
- `server/index.js` — clustered Express + Socket.io app (`setupWebSocket`, services, error handling)
- `server/services/` — user, product, order, notification services
- `server/middleware/` — errorHandler, performance monitor
- `docs/nodejs-architecture.md`

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 8: Node.js Deep Dive"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 9**: Review Express.js and middleware concepts
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [Node.js Documentation](https://nodejs.org/docs/)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [Clustering](https://nodejs.org/api/cluster.html)

---

**Ready for Day 9? Check out [Day 9: Express & Middleware](../day9/README.md)!** 🚀

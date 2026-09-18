# Day 10: MongoDB & SQL

## 🎯 Learning Objectives

- Master MongoDB document database and NoSQL concepts
- Implement PostgreSQL relational database with SQL
- Design hybrid database architecture for different use cases
- Optimize database performance with indexing and query optimization
- Implement database migrations and schema management

## 📚 Theory & Concepts

### MongoDB Fundamentals
- **Document Model**: BSON documents, collections, and databases
- **Query Language**: CRUD operations, aggregation pipeline
- **Indexing**: Single field, compound, text, and geospatial indexes
- **Schema Design**: Embedded vs referenced documents
- **Performance**: Query optimization, connection pooling

### PostgreSQL Fundamentals
- **Relational Model**: Tables, relationships, constraints
- **SQL Language**: DDL, DML, DQL, and DCL operations
- **Advanced Features**: Views, stored procedures, triggers
- **Performance**: Query optimization, indexing, partitioning
- **ACID Properties**: Atomicity, consistency, isolation, durability

### Hybrid Database Architecture
- **Use Case Analysis**: When to use MongoDB vs PostgreSQL
- **Data Modeling**: Document vs relational design patterns
- **Integration**: Connecting multiple databases
- **Consistency**: Eventual vs strong consistency
- **Scalability**: Horizontal vs vertical scaling

## 🛠️ Hands-on Tasks

### Task 1: Set Up MongoDB with Mongoose
Implement MongoDB integration with Mongoose ODM:

```javascript
// database/mongodb.js
const mongoose = require('mongoose');
const { logger } = require('../middleware/errorHandler');

class MongoDBConnection {
  constructor() {
    this.connection = null;
    this.isConnected = false;
  }

  async connect() {
    try {
      const mongoUri = process.env.MONGODB_URI || 'mongodb://localhost:27017/sda-training';
      
      this.connection = await mongoose.connect(mongoUri, {
        useNewUrlParser: true,
        useUnifiedTopology: true,
        maxPoolSize: 10,
        serverSelectionTimeoutMS: 5000,
        socketTimeoutMS: 45000,
        bufferCommands: false,
        bufferMaxEntries: 0
      });

      this.isConnected = true;
      logger.info('MongoDB connected successfully');

      // Connection event handlers
      mongoose.connection.on('error', (error) => {
        logger.error('MongoDB connection error:', error);
        this.isConnected = false;
      });

      mongoose.connection.on('disconnected', () => {
        logger.warn('MongoDB disconnected');
        this.isConnected = false;
      });

      mongoose.connection.on('reconnected', () => {
        logger.info('MongoDB reconnected');
        this.isConnected = true;
      });

    } catch (error) {
      logger.error('MongoDB connection failed:', error);
      throw error;
    }
  }

  async disconnect() {
    if (this.connection) {
      await mongoose.disconnect();
      this.isConnected = false;
      logger.info('MongoDB disconnected');
    }
  }

  getConnectionStatus() {
    return {
      isConnected: this.isConnected,
      readyState: mongoose.connection.readyState,
      host: mongoose.connection.host,
      port: mongoose.connection.port,
      name: mongoose.connection.name
    };
  }
}

module.exports = new MongoDBConnection();
```

### Task 2: Create MongoDB Models
Implement comprehensive MongoDB models with Mongoose:

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Name is required'],
    trim: true,
    maxlength: [50, 'Name cannot exceed 50 characters']
  },
  email: {
    type: String,
    required: [true, 'Email is required'],
    unique: true,
    lowercase: true,
    trim: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Please enter a valid email']
  },
  password: {
    type: String,
    required: [true, 'Password is required'],
    minlength: [8, 'Password must be at least 8 characters'],
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'moderator'],
    default: 'user'
  },
  avatar: {
    type: String,
    default: null
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: {
    type: Date,
    default: null
  },
  preferences: {
    theme: {
      type: String,
      enum: ['light', 'dark'],
      default: 'light'
    },
    notifications: {
      email: {
        type: Boolean,
        default: true
      },
      push: {
        type: Boolean,
        default: true
      }
    }
  },
  profile: {
    bio: {
      type: String,
      maxlength: [500, 'Bio cannot exceed 500 characters']
    },
    location: {
      type: String,
      maxlength: [100, 'Location cannot exceed 100 characters']
    },
    website: {
      type: String,
      match: [/^https?:\/\/.+/, 'Please enter a valid URL']
    }
  }
}, {
  timestamps: true,
  toJSON: { virtuals: true },
  toObject: { virtuals: true }
});

// Indexes
userSchema.index({ email: 1 });
userSchema.index({ role: 1 });
userSchema.index({ isActive: 1 });
userSchema.index({ createdAt: -1 });

// Virtual for full name
userSchema.virtual('fullName').get(function() {
  return this.name;
});

// Pre-save middleware to hash password
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  try {
    const salt = await bcrypt.genSalt(12);
    this.password = await bcrypt.hash(this.password, salt);
    next();
  } catch (error) {
    next(error);
  }
});

// Instance method to check password
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

// Instance method to generate JWT
userSchema.methods.generateAuthToken = function() {
  return jwt.sign(
    { 
      userId: this._id, 
      email: this.email, 
      role: this.role 
    },
    process.env.JWT_SECRET,
    { expiresIn: process.env.JWT_EXPIRES_IN || '7d' }
  );
};

// Static method to find user by credentials
userSchema.statics.findByCredentials = async function(email, password) {
  const user = await this.findOne({ email, isActive: true }).select('+password');
  if (!user) {
    throw new Error('Invalid credentials');
  }
  
  const isMatch = await user.comparePassword(password);
  if (!isMatch) {
    throw new Error('Invalid credentials');
  }
  
  return user;
};

// Static method to get user statistics
userSchema.statics.getUserStats = async function() {
  const stats = await this.aggregate([
    {
      $group: {
        _id: '$role',
        count: { $sum: 1 },
        activeUsers: {
          $sum: { $cond: [{ $eq: ['$isActive', true] }, 1, 0] }
        }
      }
    }
  ]);
  
  return stats;
};

module.exports = mongoose.model('User', userSchema);
```

### Task 3: Create PostgreSQL Connection
Implement PostgreSQL integration with connection pooling:

```javascript
// database/postgresql.js
const { Pool } = require('pg');
const { logger } = require('../middleware/errorHandler');

class PostgreSQLConnection {
  constructor() {
    this.pool = null;
    this.isConnected = false;
  }

  async connect() {
    try {
      this.pool = new Pool({
        user: process.env.POSTGRES_USER || 'postgres',
        host: process.env.POSTGRES_HOST || 'localhost',
        database: process.env.POSTGRES_DB || 'sda_training',
        password: process.env.POSTGRES_PASSWORD || 'password',
        port: process.env.POSTGRES_PORT || 5432,
        max: 20,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000,
      });

      // Test connection
      const client = await this.pool.connect();
      await client.query('SELECT NOW()');
      client.release();

      this.isConnected = true;
      logger.info('PostgreSQL connected successfully');

      // Handle pool errors
      this.pool.on('error', (err) => {
        logger.error('PostgreSQL pool error:', err);
        this.isConnected = false;
      });

    } catch (error) {
      logger.error('PostgreSQL connection failed:', error);
      throw error;
    }
  }

  async disconnect() {
    if (this.pool) {
      await this.pool.end();
      this.isConnected = false;
      logger.info('PostgreSQL disconnected');
    }
  }

  async query(text, params) {
    const start = Date.now();
    try {
      const result = await this.pool.query(text, params);
      const duration = Date.now() - start;
      logger.debug('Query executed', { text, duration, rows: result.rowCount });
      return result;
    } catch (error) {
      logger.error('Query failed', { text, error: error.message });
      throw error;
    }
  }

  async getClient() {
    return await this.pool.connect();
  }

  getConnectionStatus() {
    return {
      isConnected: this.isConnected,
      totalCount: this.pool?.totalCount || 0,
      idleCount: this.pool?.idleCount || 0,
      waitingCount: this.pool?.waitingCount || 0
    };
  }
}

module.exports = new PostgreSQLConnection();
```

### Task 4: Create PostgreSQL Models
Implement PostgreSQL models with proper relationships:

```javascript
// models/Product.js
const postgresql = require('../database/postgresql');

class Product {
  constructor() {
    this.tableName = 'products';
  }

  async create(productData) {
    const { name, description, price, category, stock, imageUrl, tags } = productData;
    
    const query = `
      INSERT INTO products (name, description, price, category, stock, image_url, tags, created_at, updated_at)
      VALUES ($1, $2, $3, $4, $5, $6, $7, NOW(), NOW())
      RETURNING *
    `;
    
    const values = [name, description, price, category, stock, imageUrl, JSON.stringify(tags)];
    const result = await postgresql.query(query, values);
    
    return result.rows[0];
  }

  async findById(id) {
    const query = `
      SELECT p.*, 
             COUNT(o.id) as order_count,
             AVG(r.rating) as average_rating
      FROM products p
      LEFT JOIN order_items oi ON p.id = oi.product_id
      LEFT JOIN orders o ON oi.order_id = o.id
      LEFT JOIN reviews r ON p.id = r.product_id
      WHERE p.id = $1
      GROUP BY p.id
    `;
    
    const result = await postgresql.query(query, [id]);
    return result.rows[0];
  }

  async findAll(filters = {}) {
    let query = `
      SELECT p.*, 
             COUNT(o.id) as order_count,
             AVG(r.rating) as average_rating
      FROM products p
      LEFT JOIN order_items oi ON p.id = oi.product_id
      LEFT JOIN orders o ON oi.order_id = o.id
      LEFT JOIN reviews r ON p.id = r.product_id
    `;
    
    const conditions = [];
    const values = [];
    let paramCount = 0;

    if (filters.category) {
      paramCount++;
      conditions.push(`p.category = $${paramCount}`);
      values.push(filters.category);
    }

    if (filters.minPrice) {
      paramCount++;
      conditions.push(`p.price >= $${paramCount}`);
      values.push(filters.minPrice);
    }

    if (filters.maxPrice) {
      paramCount++;
      conditions.push(`p.price <= $${paramCount}`);
      values.push(filters.maxPrice);
    }

    if (filters.search) {
      paramCount++;
      conditions.push(`(p.name ILIKE $${paramCount} OR p.description ILIKE $${paramCount})`);
      values.push(`%${filters.search}%`);
    }

    if (conditions.length > 0) {
      query += ` WHERE ${conditions.join(' AND ')}`;
    }

    query += ` GROUP BY p.id`;

    if (filters.sortBy) {
      const sortOrder = filters.sortOrder || 'ASC';
      query += ` ORDER BY p.${filters.sortBy} ${sortOrder}`;
    } else {
      query += ` ORDER BY p.created_at DESC`;
    }

    if (filters.limit) {
      paramCount++;
      query += ` LIMIT $${paramCount}`;
      values.push(filters.limit);
    }

    if (filters.offset) {
      paramCount++;
      query += ` OFFSET $${paramCount}`;
      values.push(filters.offset);
    }

    const result = await postgresql.query(query, values);
    return result.rows;
  }

  async update(id, updateData) {
    const fields = [];
    const values = [];
    let paramCount = 0;

    Object.keys(updateData).forEach(key => {
      if (updateData[key] !== undefined) {
        paramCount++;
        fields.push(`${key} = $${paramCount}`);
        values.push(updateData[key]);
      }
    });

    if (fields.length === 0) {
      throw new Error('No fields to update');
    }

    paramCount++;
    values.push(id);

    const query = `
      UPDATE products 
      SET ${fields.join(', ')}, updated_at = NOW()
      WHERE id = $${paramCount}
      RETURNING *
    `;

    const result = await postgresql.query(query, values);
    return result.rows[0];
  }

  async delete(id) {
    const query = 'DELETE FROM products WHERE id = $1 RETURNING *';
    const result = await postgresql.query(query, [id]);
    return result.rows[0];
  }

  async getStats() {
    const query = `
      SELECT 
        COUNT(*) as total_products,
        AVG(price) as average_price,
        MIN(price) as min_price,
        MAX(price) as max_price,
        SUM(stock) as total_stock
      FROM products
    `;
    
    const result = await postgresql.query(query);
    return result.rows[0];
  }

  async getCategoryStats() {
    const query = `
      SELECT 
        category,
        COUNT(*) as product_count,
        AVG(price) as average_price,
        SUM(stock) as total_stock
      FROM products
      GROUP BY category
      ORDER BY product_count DESC
    `;
    
    const result = await postgresql.query(query);
    return result.rows;
  }
}

module.exports = new Product();
```

### Task 5: Create Database Migrations
Implement database migrations for schema management:

```javascript
// migrations/001_create_users_table.js
const postgresql = require('../database/postgresql');

async function up() {
  const query = `
    CREATE TABLE IF NOT EXISTS users (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      name VARCHAR(100) NOT NULL,
      email VARCHAR(255) UNIQUE NOT NULL,
      password_hash VARCHAR(255) NOT NULL,
      role VARCHAR(20) DEFAULT 'user' CHECK (role IN ('user', 'admin', 'moderator')),
      avatar_url VARCHAR(500),
      is_active BOOLEAN DEFAULT true,
      last_login TIMESTAMP,
      preferences JSONB DEFAULT '{}',
      profile JSONB DEFAULT '{}',
      created_at TIMESTAMP DEFAULT NOW(),
      updated_at TIMESTAMP DEFAULT NOW()
    );

    CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
    CREATE INDEX IF NOT EXISTS idx_users_role ON users(role);
    CREATE INDEX IF NOT EXISTS idx_users_is_active ON users(is_active);
    CREATE INDEX IF NOT EXISTS idx_users_created_at ON users(created_at);
  `;

  await postgresql.query(query);
  console.log('Users table created successfully');
}

async function down() {
  const query = 'DROP TABLE IF EXISTS users CASCADE';
  await postgresql.query(query);
  console.log('Users table dropped successfully');
}

module.exports = { up, down };
```

## 📝 Documentation Tasks

### Create Database Architecture Guide
Create `week2/day10/docs/database-architecture.md`:

```markdown
# Database Architecture Guide

## Hybrid Database Strategy
- **MongoDB**: Document storage for flexible schemas
- **PostgreSQL**: Relational data with ACID properties
- **Use Case Analysis**: When to use each database
- **Data Consistency**: Eventual vs strong consistency
- **Performance**: Query optimization and indexing

## MongoDB Best Practices
- **Schema Design**: Embedded vs referenced documents
- **Indexing**: Single field, compound, and text indexes
- **Query Optimization**: Aggregation pipeline and performance
- **Connection Management**: Connection pooling and monitoring
- **Data Modeling**: Document structure and relationships

## PostgreSQL Best Practices
- **Schema Design**: Normalization and relationships
- **Indexing**: B-tree, hash, and specialized indexes
- **Query Optimization**: EXPLAIN ANALYZE and performance tuning
- **Connection Management**: Connection pooling and monitoring
- **Data Integrity**: Constraints and triggers
```

## 🧪 Testing & Validation

### Database Testing
- [x] MongoDB connection works correctly
- [x] PostgreSQL connection works correctly
- [x] Models work correctly
- [x] Migrations work correctly
- [x] Performance is acceptable

### Data Integrity Testing
- [x] Data is stored correctly
- [x] Relationships work correctly
- [x] Constraints work correctly
- [x] Indexes work correctly
- [x] Queries are optimized

## 📊 Success Criteria

By the end of Day 10, you should have:

✅ **MongoDB Mastery**: Document database implementation  
✅ **PostgreSQL Mastery**: Relational database implementation  
✅ **Hybrid Architecture**: Multi-database integration  
✅ **Performance**: Optimized queries and indexing  
✅ **Data Integrity**: Proper constraints and relationships  

## ✅ Completion Status

All Day 10 deliverables are complete and tested:
- `server/database/mongodb.js`, `server/database/postgresql.js` — dual-database layer
- `server/migrations/` — users/products/orders tables + `migrate.js` runner
- `server/routes/` + `server/services/` + 5 middleware (auth, errorHandler, logger, performance, validation)
- `docs/database-architecture.md`

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 10: MongoDB & SQL"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 11**: Review REST API best practices
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [MongoDB Documentation](https://docs.mongodb.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Mongoose ODM](https://mongoosejs.com/)
- [Node.js PostgreSQL](https://node-postgres.com/)

---

**Ready for Day 11? Check out [Day 11: REST API Best Practices](../day11/README.md)!** 🚀

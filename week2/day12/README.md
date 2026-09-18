# Day 12: Authentication & RBAC

## 🎯 Learning Objectives

- Master JWT authentication and token management
- Implement OAuth2 and social authentication
- Build role-based access control (RBAC) system
- Create secure password policies and validation
- Implement session management and security

## 📚 Theory & Concepts

### Authentication Methods
- **JWT Tokens**: JSON Web Tokens for stateless authentication
- **OAuth2**: Authorization framework for third-party access
- **Social Login**: Google, Facebook, GitHub authentication
- **Session Management**: Server-side session storage
- **Multi-Factor Authentication**: 2FA and MFA implementation

### Authorization Patterns
- **RBAC**: Role-based access control with permissions
- **ABAC**: Attribute-based access control
- **ACL**: Access control lists
- **Policy-Based**: Rule-based authorization
- **Hierarchical**: Role inheritance and delegation

### Security Best Practices
- **Password Security**: Hashing, salting, and strength requirements
- **Token Security**: Secure storage, rotation, and expiration
- **Input Validation**: Sanitization and validation
- **Rate Limiting**: Brute force protection
- **Audit Logging**: Security event tracking

## 🛠️ Hands-on Tasks

### Task 1: Implement JWT Authentication
Create comprehensive JWT authentication system:

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const { AppError } = require('./errorHandler');
const User = require('../models/User');

class AuthService {
  constructor() {
    this.jwtSecret = process.env.JWT_SECRET || 'your-secret-key';
    this.jwtExpiresIn = process.env.JWT_EXPIRES_IN || '7d';
    this.refreshTokenExpiresIn = process.env.REFRESH_TOKEN_EXPIRES_IN || '30d';
  }

  async generateTokens(user) {
    const payload = {
      userId: user._id,
      email: user.email,
      role: user.role
    };

    const accessToken = jwt.sign(payload, this.jwtSecret, {
      expiresIn: this.jwtExpiresIn,
      issuer: 'sda-training-api',
      audience: 'sda-training-client'
    });

    const refreshToken = jwt.sign(
      { userId: user._id, type: 'refresh' },
      this.jwtSecret,
      { expiresIn: this.refreshTokenExpiresIn }
    );

    return { accessToken, refreshToken };
  }

  async verifyToken(token) {
    try {
      const decoded = jwt.verify(token, this.jwtSecret, {
        issuer: 'sda-training-api',
        audience: 'sda-training-client'
      });
      return decoded;
    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        throw new AppError('Token expired', 401);
      } else if (error.name === 'JsonWebTokenError') {
        throw new AppError('Invalid token', 401);
      }
      throw error;
    }
  }

  async refreshToken(refreshToken) {
    try {
      const decoded = jwt.verify(refreshToken, this.jwtSecret);
      
      if (decoded.type !== 'refresh') {
        throw new AppError('Invalid refresh token', 401);
      }

      const user = await User.findById(decoded.userId);
      if (!user || !user.isActive) {
        throw new AppError('User not found or inactive', 401);
      }

      return await this.generateTokens(user);
    } catch (error) {
      throw new AppError('Invalid refresh token', 401);
    }
  }

  async hashPassword(password) {
    const saltRounds = 12;
    return await bcrypt.hash(password, saltRounds);
  }

  async comparePassword(password, hashedPassword) {
    return await bcrypt.compare(password, hashedPassword);
  }

  async validatePassword(password) {
    const minLength = 8;
    const hasUpperCase = /[A-Z]/.test(password);
    const hasLowerCase = /[a-z]/.test(password);
    const hasNumbers = /\d/.test(password);
    const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(password);

    const errors = [];
    
    if (password.length < minLength) {
      errors.push(`Password must be at least ${minLength} characters long`);
    }
    if (!hasUpperCase) {
      errors.push('Password must contain at least one uppercase letter');
    }
    if (!hasLowerCase) {
      errors.push('Password must contain at least one lowercase letter');
    }
    if (!hasNumbers) {
      errors.push('Password must contain at least one number');
    }
    if (!hasSpecialChar) {
      errors.push('Password must contain at least one special character');
    }

    if (errors.length > 0) {
      throw new AppError('Password validation failed', 400, errors);
    }

    return true;
  }
}

const authService = new AuthService();

// Authentication middleware
const authenticate = async (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new AppError('Access token is required', 401);
    }

    const token = authHeader.substring(7);
    const decoded = await authService.verifyToken(token);
    
    const user = await User.findById(decoded.userId);
    if (!user || !user.isActive) {
      throw new AppError('User not found or inactive', 401);
    }

    req.user = user;
    next();
  } catch (error) {
    next(error);
  }
};

// Authorization middleware
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

// Optional authentication
const optionalAuth = async (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (authHeader && authHeader.startsWith('Bearer ')) {
      const token = authHeader.substring(7);
      const decoded = await authService.verifyToken(token);
      const user = await User.findById(decoded.userId);
      
      if (user && user.isActive) {
        req.user = user;
      }
    }
    
    next();
  } catch (error) {
    // Continue without authentication
    next();
  }
};

module.exports = {
  authService,
  authenticate,
  authorize,
  optionalAuth
};
```

### Task 2: Implement OAuth2 and Social Login
Create OAuth2 and social authentication system:

```javascript
// middleware/oauth.js
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;
const FacebookStrategy = require('passport-facebook').Strategy;
const GitHubStrategy = require('passport-github2').Strategy;
const User = require('../models/User');
const { authService } = require('./auth');

// Google OAuth2 Strategy
passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  callbackURL: '/api/v1/auth/google/callback'
}, async (accessToken, refreshToken, profile, done) => {
  try {
    const existingUser = await User.findOne({ 
      $or: [
        { googleId: profile.id },
        { email: profile.emails[0].value }
      ]
    });

    if (existingUser) {
      if (!existingUser.googleId) {
        existingUser.googleId = profile.id;
        await existingUser.save();
      }
      return done(null, existingUser);
    }

    const newUser = new User({
      googleId: profile.id,
      name: profile.displayName,
      email: profile.emails[0].value,
      avatar: profile.photos[0].value,
      isActive: true,
      role: 'user'
    });

    await newUser.save();
    done(null, newUser);
  } catch (error) {
    done(error, null);
  }
}));

// Facebook Strategy
passport.use(new FacebookStrategy({
  clientID: process.env.FACEBOOK_APP_ID,
  clientSecret: process.env.FACEBOOK_APP_SECRET,
  callbackURL: '/api/v1/auth/facebook/callback',
  profileFields: ['id', 'emails', 'name', 'picture']
}, async (accessToken, refreshToken, profile, done) => {
  try {
    const existingUser = await User.findOne({ 
      $or: [
        { facebookId: profile.id },
        { email: profile.emails[0].value }
      ]
    });

    if (existingUser) {
      if (!existingUser.facebookId) {
        existingUser.facebookId = profile.id;
        await existingUser.save();
      }
      return done(null, existingUser);
    }

    const newUser = new User({
      facebookId: profile.id,
      name: `${profile.name.givenName} ${profile.name.familyName}`,
      email: profile.emails[0].value,
      avatar: profile.photos[0].value,
      isActive: true,
      role: 'user'
    });

    await newUser.save();
    done(null, newUser);
  } catch (error) {
    done(error, null);
  }
}));

// GitHub Strategy
passport.use(new GitHubStrategy({
  clientID: process.env.GITHUB_CLIENT_ID,
  clientSecret: process.env.GITHUB_CLIENT_SECRET,
  callbackURL: '/api/v1/auth/github/callback'
}, async (accessToken, refreshToken, profile, done) => {
  try {
    const existingUser = await User.findOne({ 
      $or: [
        { githubId: profile.id },
        { email: profile.emails[0].value }
      ]
    });

    if (existingUser) {
      if (!existingUser.githubId) {
        existingUser.githubId = profile.id;
        await existingUser.save();
      }
      return done(null, existingUser);
    }

    const newUser = new User({
      githubId: profile.id,
      name: profile.displayName,
      email: profile.emails[0].value,
      avatar: profile.photos[0].value,
      isActive: true,
      role: 'user'
    });

    await newUser.save();
    done(null, newUser);
  } catch (error) {
    done(error, null);
  }
}));

// Serialize user for session
passport.serializeUser((user, done) => {
  done(null, user._id);
});

// Deserialize user from session
passport.deserializeUser(async (id, done) => {
  try {
    const user = await User.findById(id);
    done(null, user);
  } catch (error) {
    done(error, null);
  }
});

module.exports = passport;
```

### Task 3: Implement RBAC System
Create comprehensive role-based access control:

```javascript
// middleware/rbac.js
const { AppError } = require('./errorHandler');

// Permission definitions
const PERMISSIONS = {
  // User permissions
  'users:read': ['admin', 'moderator'],
  'users:write': ['admin'],
  'users:delete': ['admin'],
  
  // Product permissions
  'products:read': ['admin', 'moderator', 'user'],
  'products:write': ['admin', 'moderator'],
  'products:delete': ['admin'],
  
  // Order permissions
  'orders:read': ['admin', 'moderator', 'user'],
  'orders:write': ['admin', 'moderator', 'user'],
  'orders:delete': ['admin'],
  
  // Analytics permissions
  'analytics:read': ['admin', 'moderator'],
  'analytics:write': ['admin'],
  
  // System permissions
  'system:read': ['admin'],
  'system:write': ['admin'],
  'system:delete': ['admin']
};

// Check if user has permission
const hasPermission = (user, permission) => {
  if (!user || !user.role) return false;
  
  const allowedRoles = PERMISSIONS[permission];
  if (!allowedRoles) return false;
  
  return allowedRoles.includes(user.role);
};

// Permission middleware
const requirePermission = (permission) => {
  return (req, res, next) => {
    if (!req.user) {
      return next(new AppError('Authentication required', 401));
    }

    if (!hasPermission(req.user, permission)) {
      return next(new AppError('Insufficient permissions', 403));
    }

    next();
  };
};

// Multiple permissions middleware
const requireAnyPermission = (...permissions) => {
  return (req, res, next) => {
    if (!req.user) {
      return next(new AppError('Authentication required', 401));
    }

    const hasAnyPermission = permissions.some(permission => 
      hasPermission(req.user, permission)
    );

    if (!hasAnyPermission) {
      return next(new AppError('Insufficient permissions', 403));
    }

    next();
  };
};

// All permissions middleware
const requireAllPermissions = (...permissions) => {
  return (req, res, next) => {
    if (!req.user) {
      return next(new AppError('Authentication required', 401));
    }

    const hasAllPermissions = permissions.every(permission => 
      hasPermission(req.user, permission)
    );

    if (!hasAllPermissions) {
      return next(new AppError('Insufficient permissions', 403));
    }

    next();
  };
};

// Resource ownership middleware
const requireOwnership = (resourceField = 'userId') => {
  return (req, res, next) => {
    if (!req.user) {
      return next(new AppError('Authentication required', 401));
    }

    // Admin can access any resource
    if (req.user.role === 'admin') {
      return next();
    }

    // Check if user owns the resource
    const resourceUserId = req.params[resourceField] || req.body[resourceField];
    if (resourceUserId && resourceUserId !== req.user._id.toString()) {
      return next(new AppError('Access denied: insufficient permissions', 403));
    }

    next();
  };
};

// Role hierarchy middleware
const requireRole = (...roles) => {
  return (req, res, next) => {
    if (!req.user) {
      return next(new AppError('Authentication required', 401));
    }

    if (!roles.includes(req.user.role)) {
      return next(new AppError('Insufficient role permissions', 403));
    }

    next();
  };
};

module.exports = {
  PERMISSIONS,
  hasPermission,
  requirePermission,
  requireAnyPermission,
  requireAllPermissions,
  requireOwnership,
  requireRole
};
```

### Task 4: Create Authentication Routes
Implement comprehensive authentication routes:

```javascript
// routes/authRoutes.js
const express = require('express');
const router = express.Router();
const passport = require('../middleware/oauth');
const { authService } = require('../middleware/auth');
const { authenticate, authorize } = require('../middleware/auth');
const { requirePermission } = require('../middleware/rbac');
const User = require('../models/User');
const { AppError } = require('../middleware/errorHandler');

// Register
router.post('/register', async (req, res, next) => {
  try {
    const { name, email, password, role = 'user' } = req.body;

    // Validate password
    await authService.validatePassword(password);

    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      throw new AppError('User already exists', 400);
    }

    // Hash password
    const hashedPassword = await authService.hashPassword(password);

    // Create user
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role
    });

    await user.save();

    // Generate tokens
    const tokens = await authService.generateTokens(user);

    res.status(201).json({
      success: true,
      message: 'User registered successfully',
      data: {
        user: {
          id: user._id,
          name: user.name,
          email: user.email,
          role: user.role
        },
        ...tokens
      }
    });
  } catch (error) {
    next(error);
  }
});

// Login
router.post('/login', async (req, res, next) => {
  try {
    const { email, password } = req.body;

    // Find user
    const user = await User.findOne({ email, isActive: true }).select('+password');
    if (!user) {
      throw new AppError('Invalid credentials', 401);
    }

    // Check password
    const isPasswordValid = await authService.comparePassword(password, user.password);
    if (!isPasswordValid) {
      throw new AppError('Invalid credentials', 401);
    }

    // Update last login
    user.lastLogin = new Date();
    await user.save();

    // Generate tokens
    const tokens = await authService.generateTokens(user);

    res.json({
      success: true,
      message: 'Login successful',
      data: {
        user: {
          id: user._id,
          name: user.name,
          email: user.email,
          role: user.role
        },
        ...tokens
      }
    });
  } catch (error) {
    next(error);
  }
});

// Refresh token
router.post('/refresh', async (req, res, next) => {
  try {
    const { refreshToken } = req.body;

    if (!refreshToken) {
      throw new AppError('Refresh token is required', 400);
    }

    const tokens = await authService.refreshToken(refreshToken);

    res.json({
      success: true,
      message: 'Token refreshed successfully',
      data: tokens
    });
  } catch (error) {
    next(error);
  }
});

// Logout
router.post('/logout', authenticate, async (req, res, next) => {
  try {
    // In a real application, you would blacklist the token
    // For now, we'll just return success
    res.json({
      success: true,
      message: 'Logged out successfully'
    });
  } catch (error) {
    next(error);
  }
});

// Google OAuth
router.get('/google', passport.authenticate('google', {
  scope: ['profile', 'email']
}));

router.get('/google/callback', 
  passport.authenticate('google', { session: false }),
  async (req, res, next) => {
    try {
      const tokens = await authService.generateTokens(req.user);
      
      res.json({
        success: true,
        message: 'Google authentication successful',
        data: {
          user: {
            id: req.user._id,
            name: req.user.name,
            email: req.user.email,
            role: req.user.role
          },
          ...tokens
        }
      });
    } catch (error) {
      next(error);
    }
  }
);

// Facebook OAuth
router.get('/facebook', passport.authenticate('facebook', {
  scope: ['email']
}));

router.get('/facebook/callback',
  passport.authenticate('facebook', { session: false }),
  async (req, res, next) => {
    try {
      const tokens = await authService.generateTokens(req.user);
      
      res.json({
        success: true,
        message: 'Facebook authentication successful',
        data: {
          user: {
            id: req.user._id,
            name: req.user.name,
            email: req.user.email,
            role: req.user.role
          },
          ...tokens
        }
      });
    } catch (error) {
      next(error);
    }
  }
);

// GitHub OAuth
router.get('/github', passport.authenticate('github', {
  scope: ['user:email']
}));

router.get('/github/callback',
  passport.authenticate('github', { session: false }),
  async (req, res, next) => {
    try {
      const tokens = await authService.generateTokens(req.user);
      
      res.json({
        success: true,
        message: 'GitHub authentication successful',
        data: {
          user: {
            id: req.user._id,
            name: req.user.name,
            email: req.user.email,
            role: req.user.role
          },
          ...tokens
        }
      });
    } catch (error) {
      next(error);
    }
  }
);

// Get current user
router.get('/me', authenticate, async (req, res, next) => {
  try {
    res.json({
      success: true,
      data: {
        id: req.user._id,
        name: req.user.name,
        email: req.user.email,
        role: req.user.role,
        avatar: req.user.avatar,
        isActive: req.user.isActive,
        lastLogin: req.user.lastLogin,
        createdAt: req.user.createdAt
      }
    });
  } catch (error) {
    next(error);
  }
});

// Change password
router.put('/change-password', authenticate, async (req, res, next) => {
  try {
    const { currentPassword, newPassword } = req.body;

    // Validate current password
    const isCurrentPasswordValid = await authService.comparePassword(
      currentPassword, 
      req.user.password
    );
    if (!isCurrentPasswordValid) {
      throw new AppError('Current password is incorrect', 400);
    }

    // Validate new password
    await authService.validatePassword(newPassword);

    // Hash new password
    const hashedNewPassword = await authService.hashPassword(newPassword);

    // Update password
    req.user.password = hashedNewPassword;
    await req.user.save();

    res.json({
      success: true,
      message: 'Password changed successfully'
    });
  } catch (error) {
    next(error);
  }
});

module.exports = router;
```

## 📝 Documentation Tasks

### Create Authentication Guide
Create `week2/day12/docs/authentication-guide.md`:

```markdown
# Authentication & Authorization Guide

## Authentication Methods
- **JWT Tokens**: Stateless authentication with access and refresh tokens
- **OAuth2**: Third-party authentication with Google, Facebook, GitHub
- **Social Login**: Seamless user experience with social providers
- **Session Management**: Server-side session storage and management
- **Multi-Factor Authentication**: 2FA and MFA for enhanced security

## Authorization Patterns
- **RBAC**: Role-based access control with permissions
- **Resource Ownership**: User-specific resource access
- **Permission System**: Granular permission management
- **Role Hierarchy**: Admin, moderator, and user roles
- **Access Control**: Fine-grained access control

## Security Best Practices
- **Password Security**: Strong password requirements and hashing
- **Token Security**: Secure token storage and rotation
- **Input Validation**: Comprehensive input sanitization
- **Rate Limiting**: Brute force protection and throttling
- **Audit Logging**: Security event tracking and monitoring
```

## 🧪 Testing & Validation

### Authentication Testing
- [x] JWT authentication works correctly
- [x] OAuth2 authentication works
- [x] Password validation works
- [x] Token refresh works
- [x] Logout works correctly

### Authorization Testing
- [x] RBAC system works correctly
- [x] Permission checks work
- [x] Role hierarchy works
- [x] Resource ownership works
- [x] Access control works

## 📊 Success Criteria

By the end of Day 12, you should have:

✅ **JWT Mastery**: Secure token-based authentication  
✅ **OAuth2 Implementation**: Third-party authentication  
✅ **RBAC System**: Role-based access control  
✅ **Security**: Comprehensive security measures  
✅ **Authorization**: Fine-grained permission system  

## ✅ Completion Status

All Day 12 deliverables are complete and tested:
- `server/middleware/auth.js`, `oauth.js`, `rbac.js` — JWT + OAuth2 + role-based access control
- `server/routes/authRoutes.js` + versioned `api/v1/*` routes
- Full middleware stack (rateLimiting, caching, apiVersioning, validation, performance, logger, errorHandler)
- `docs/authentication-guide.md` (+ `api-design.md`, `swagger.js`)

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 12: Authentication & RBAC"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 13**: Review API documentation and testing
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [JWT Authentication](https://jwt.io/)
- [OAuth2 Guide](https://oauth.net/2/)
- [Passport.js](http://www.passportjs.org/)
- [RBAC Patterns](https://en.wikipedia.org/wiki/Role-based_access_control)

---

**Ready for Day 13? Check out [Day 13: API Documentation](../day13/README.md)!** 🚀

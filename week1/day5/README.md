# Day 5: API & Real-Time Data

## 🎯 Learning Objectives

- Master REST API consumption and error handling
- Implement WebSocket connections for real-time data
- Build data fetching strategies with caching and optimization
- Create real-time data visualization with live updates
- Handle API rate limiting and retry mechanisms

## 📚 Theory & Concepts

### REST API Best Practices
- **HTTP Methods**: GET, POST, PUT, DELETE, PATCH
- **Status Codes**: 200, 201, 400, 401, 403, 404, 500
- **Headers**: Content-Type, Authorization, Cache-Control
- **Error Handling**: Try-catch, retry logic, fallback strategies
- **Rate Limiting**: Request throttling, exponential backoff

### WebSocket Implementation
- **Connection Management**: Connect, disconnect, reconnect
- **Message Handling**: JSON parsing, event types
- **Error Recovery**: Connection drops, network issues
- **Performance**: Message queuing, batch processing
- **Security**: Authentication, message validation

### Data Fetching Strategies
- **Caching**: Browser cache, memory cache, localStorage
- **Optimization**: Request deduplication, parallel requests
- **Loading States**: Skeleton screens, progress indicators
- **Error States**: Retry buttons, fallback content
- **Real-time Updates**: WebSocket integration, data synchronization

## 🛠️ Hands-on Tasks

### Task 1: Create API Service Layer
Build a comprehensive API service with error handling and caching:

```javascript
// src/services/ApiService.js
class ApiService {
  constructor(baseURL, options = {}) {
    this.baseURL = baseURL;
    this.cache = new Map();
    this.retryAttempts = options.retryAttempts || 3;
    this.retryDelay = options.retryDelay || 1000;
    this.timeout = options.timeout || 10000;
    this.subscribers = new Set();
  }

  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const cacheKey = `${url}-${JSON.stringify(options)}`;
    
    // Check cache first
    if (options.cache !== false && this.cache.has(cacheKey)) {
      const cached = this.cache.get(cacheKey);
      if (Date.now() - cached.timestamp < (options.cacheTTL || 300000)) {
        return cached.data;
      }
    }

    const config = {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      },
      timeout: this.timeout,
      ...options
    };

    try {
      const response = await this.fetchWithRetry(url, config);
      const data = await response.json();
      
      // Cache successful responses
      if (options.cache !== false) {
        this.cache.set(cacheKey, {
          data,
          timestamp: Date.now()
        });
      }
      
      return data;
    } catch (error) {
      console.error('API request failed:', error);
      throw error;
    }
  }

  async fetchWithRetry(url, config, attempt = 1) {
    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), this.timeout);
      
      const response = await fetch(url, {
        ...config,
        signal: controller.signal
      });
      
      clearTimeout(timeoutId);
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return response;
    } catch (error) {
      if (attempt < this.retryAttempts && this.shouldRetry(error)) {
        await this.delay(this.retryDelay * Math.pow(2, attempt - 1));
        return this.fetchWithRetry(url, config, attempt + 1);
      }
      throw error;
    }
  }

  shouldRetry(error) {
    return error.name === 'AbortError' || 
           error.message.includes('500') || 
           error.message.includes('502') || 
           error.message.includes('503');
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  // CRUD operations
  async get(endpoint, options = {}) {
    return this.request(endpoint, { ...options, method: 'GET' });
  }

  async post(endpoint, data, options = {}) {
    return this.request(endpoint, {
      ...options,
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  async put(endpoint, data, options = {}) {
    return this.request(endpoint, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(data)
    });
  }

  async delete(endpoint, options = {}) {
    return this.request(endpoint, { ...options, method: 'DELETE' });
  }

  // Cache management
  clearCache() {
    this.cache.clear();
  }

  getCacheSize() {
    return this.cache.size;
  }

  // Subscriber management
  subscribe(callback) {
    this.subscribers.add(callback);
    return () => this.subscribers.delete(callback);
  }

  notifySubscribers(event, data) {
    this.subscribers.forEach(callback => callback(event, data));
  }
}

export default ApiService;
```

### Task 2: Implement WebSocket Service
Create a robust WebSocket service with reconnection logic:

```javascript
// src/services/WebSocketService.js
class WebSocketService {
  constructor(url, options = {}) {
    this.url = url;
    this.options = {
      reconnectInterval: 5000,
      maxReconnectAttempts: 10,
      heartbeatInterval: 30000,
      ...options
    };
    this.ws = null;
    this.reconnectAttempts = 0;
    this.heartbeatTimer = null;
    this.subscribers = new Map();
    this.messageQueue = [];
    this.isConnected = false;
  }

  connect() {
    try {
      this.ws = new WebSocket(this.url);
      this.setupEventListeners();
    } catch (error) {
      console.error('WebSocket connection failed:', error);
      this.handleReconnect();
    }
  }

  setupEventListeners() {
    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.isConnected = true;
      this.reconnectAttempts = 0;
      this.startHeartbeat();
      this.processMessageQueue();
      this.notifySubscribers('connected', null);
    };

    this.ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        this.handleMessage(data);
      } catch (error) {
        console.error('Failed to parse WebSocket message:', error);
      }
    };

    this.ws.onclose = (event) => {
      console.log('WebSocket disconnected:', event.code, event.reason);
      this.isConnected = false;
      this.stopHeartbeat();
      this.notifySubscribers('disconnected', { code: event.code, reason: event.reason });
      
      if (!event.wasClean) {
        this.handleReconnect();
      }
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.notifySubscribers('error', error);
    };
  }

  handleMessage(data) {
    const { type, payload } = data;
    
    if (type === 'pong') {
      return; // Heartbeat response
    }
    
    this.notifySubscribers('message', { type, payload });
    
    // Notify specific subscribers
    if (this.subscribers.has(type)) {
      this.subscribers.get(type).forEach(callback => callback(payload));
    }
  }

  send(data) {
    if (this.isConnected && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    } else {
      this.messageQueue.push(data);
    }
  }

  subscribe(eventType, callback) {
    if (!this.subscribers.has(eventType)) {
      this.subscribers.set(eventType, new Set());
    }
    this.subscribers.get(eventType).add(callback);
    
    return () => {
      if (this.subscribers.has(eventType)) {
        this.subscribers.get(eventType).delete(callback);
      }
    };
  }

  notifySubscribers(event, data) {
    if (this.subscribers.has(event)) {
      this.subscribers.get(event).forEach(callback => callback(data));
    }
  }

  startHeartbeat() {
    this.heartbeatTimer = setInterval(() => {
      if (this.isConnected) {
        this.send({ type: 'ping' });
      }
    }, this.options.heartbeatInterval);
  }

  stopHeartbeat() {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  processMessageQueue() {
    while (this.messageQueue.length > 0 && this.isConnected) {
      const message = this.messageQueue.shift();
      this.send(message);
    }
  }

  handleReconnect() {
    if (this.reconnectAttempts < this.options.maxReconnectAttempts) {
      this.reconnectAttempts++;
      console.log(`Attempting to reconnect (${this.reconnectAttempts}/${this.options.maxReconnectAttempts})`);
      
      setTimeout(() => {
        this.connect();
      }, this.options.reconnectInterval);
    } else {
      console.error('Max reconnection attempts reached');
      this.notifySubscribers('maxReconnectAttemptsReached', null);
    }
  }

  disconnect() {
    this.stopHeartbeat();
    if (this.ws) {
      this.ws.close(1000, 'Client disconnect');
    }
    this.isConnected = false;
  }

  getConnectionState() {
    return {
      isConnected: this.isConnected,
      reconnectAttempts: this.reconnectAttempts,
      url: this.url
    };
  }
}

export default WebSocketService;
```

### Task 3: Create Real-Time Data Hook
Implement a custom hook for real-time data management:

```javascript
// src/hooks/useRealTimeData.js
import { useState, useEffect, useCallback, useRef } from 'react';
import { useWebSocket } from './useWebSocket';
import { useApiService } from './useApiService';

export function useRealTimeData(endpoint, options = {}) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [isConnected, setIsConnected] = useState(false);
  
  const apiService = useApiService();
  const wsService = useWebSocket(options.wsUrl);
  const lastUpdateRef = useRef(null);

  // Initial data fetch
  useEffect(() => {
    const fetchInitialData = async () => {
      try {
        setLoading(true);
        const initialData = await apiService.get(endpoint);
        setData(initialData);
        setError(null);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchInitialData();
  }, [endpoint, apiService]);

  // WebSocket connection management
  useEffect(() => {
    if (options.enableRealTime) {
      wsService.connect();
    }

    return () => {
      if (options.enableRealTime) {
        wsService.disconnect();
      }
    };
  }, [options.enableRealTime, wsService]);

  // Handle WebSocket messages
  useEffect(() => {
    if (!options.enableRealTime) return;

    const handleMessage = (message) => {
      const { type, payload } = message;
      
      if (type === 'dataUpdate' && payload.endpoint === endpoint) {
        setData(prevData => ({
          ...prevData,
          ...payload.data
        }));
        lastUpdateRef.current = Date.now();
      }
    };

    const unsubscribe = wsService.subscribe('message', handleMessage);
    return unsubscribe;
  }, [endpoint, options.enableRealTime, wsService]);

  // Connection status
  useEffect(() => {
    const handleConnectionChange = (status) => {
      setIsConnected(status === 'connected');
    };

    const unsubscribe = wsService.subscribe('connected', handleConnectionChange);
    return unsubscribe;
  }, [wsService]);

  // Manual refresh
  const refresh = useCallback(async () => {
    try {
      setLoading(true);
      const freshData = await apiService.get(endpoint);
      setData(freshData);
      setError(null);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [endpoint, apiService]);

  // Get last update time
  const getLastUpdate = useCallback(() => {
    return lastUpdateRef.current;
  }, []);

  return {
    data,
    loading,
    error,
    isConnected,
    refresh,
    getLastUpdate
  };
}
```

### Task 4: Create Real-Time Dashboard
Build a dashboard that displays real-time data with live updates:

```jsx
// src/components/RealTimeDashboard.jsx
import React, { useState, useEffect } from 'react';
import { useRealTimeData } from '../hooks/useRealTimeData';
import { MetricsCard } from './MetricsCard';
import { ChartContainer } from './ChartContainer';
import { ConnectionStatus } from './ConnectionStatus';
import './RealTimeDashboard.css';

export function RealTimeDashboard() {
  const [selectedMetric, setSelectedMetric] = useState('revenue');
  const [autoRefresh, setAutoRefresh] = useState(true);
  
  const {
    data: revenueData,
    loading: revenueLoading,
    error: revenueError,
    isConnected: revenueConnected,
    refresh: refreshRevenue
  } = useRealTimeData('/api/revenue', { enableRealTime: true });

  const {
    data: userData,
    loading: userLoading,
    error: userError,
    isConnected: userConnected,
    refresh: refreshUsers
  } = useRealTimeData('/api/users', { enableRealTime: true });

  const {
    data: orderData,
    loading: orderLoading,
    error: orderError,
    isConnected: orderConnected,
    refresh: refreshOrders
  } = useRealTimeData('/api/orders', { enableRealTime: true });

  // Auto-refresh logic
  useEffect(() => {
    if (!autoRefresh) return;

    const interval = setInterval(() => {
      if (revenueConnected) refreshRevenue();
      if (userConnected) refreshUsers();
      if (orderConnected) refreshOrders();
    }, 30000); // Refresh every 30 seconds

    return () => clearInterval(interval);
  }, [autoRefresh, revenueConnected, userConnected, orderConnected, refreshRevenue, refreshUsers, refreshOrders]);

  const handleRefreshAll = () => {
    refreshRevenue();
    refreshUsers();
    refreshOrders();
  };

  const getConnectionStatus = () => {
    const connections = [revenueConnected, userConnected, orderConnected];
    const connectedCount = connections.filter(Boolean).length;
    
    if (connectedCount === 0) return 'disconnected';
    if (connectedCount === connections.length) return 'connected';
    return 'partial';
  };

  if (revenueLoading && userLoading && orderLoading) {
    return <LoadingSpinner />;
  }

  return (
    <div className="real-time-dashboard">
      <div className="dashboard-header">
        <h1>Real-Time Dashboard</h1>
        <div className="dashboard-controls">
          <ConnectionStatus status={getConnectionStatus()} />
          <button 
            className="refresh-btn"
            onClick={handleRefreshAll}
            disabled={revenueLoading || userLoading || orderLoading}
          >
            Refresh All
          </button>
          <label className="auto-refresh-toggle">
            <input
              type="checkbox"
              checked={autoRefresh}
              onChange={(e) => setAutoRefresh(e.target.checked)}
            />
            Auto Refresh
          </label>
        </div>
      </div>

      <div className="metrics-grid">
        <MetricsCard
          title="Revenue"
          value={revenueData?.total || 0}
          change={revenueData?.change || 0}
          loading={revenueLoading}
          error={revenueError}
          connected={revenueConnected}
          onRefresh={refreshRevenue}
        />
        <MetricsCard
          title="Users"
          value={userData?.total || 0}
          change={userData?.change || 0}
          loading={userLoading}
          error={userError}
          connected={userConnected}
          onRefresh={refreshUsers}
        />
        <MetricsCard
          title="Orders"
          value={orderData?.total || 0}
          change={orderData?.change || 0}
          loading={orderLoading}
          error={orderError}
          connected={orderConnected}
          onRefresh={refreshOrders}
        />
      </div>

      <div className="charts-section">
        <ChartContainer
          data={{
            revenue: revenueData,
            users: userData,
            orders: orderData
          }}
          selectedMetric={selectedMetric}
          onMetricChange={setSelectedMetric}
          realTime={true}
        />
      </div>
    </div>
  );
}

function LoadingSpinner() {
  return (
    <div className="loading-container">
      <div className="spinner"></div>
      <p>Loading real-time data...</p>
    </div>
  );
}
```

### Task 5: Create Connection Status Component
Build a component to show connection status and handle reconnection:

```jsx
// src/components/ConnectionStatus.jsx
import React, { useState, useEffect } from 'react';

export function ConnectionStatus({ status, onReconnect }) {
  const [showDetails, setShowDetails] = useState(false);
  const [lastUpdate, setLastUpdate] = useState(null);

  useEffect(() => {
    if (status === 'connected') {
      setLastUpdate(new Date());
    }
  }, [status]);

  const getStatusInfo = () => {
    switch (status) {
      case 'connected':
        return {
          icon: '🟢',
          text: 'Connected',
          color: '#10b981',
          description: 'All services are connected and receiving real-time updates'
        };
      case 'partial':
        return {
          icon: '🟡',
          text: 'Partial Connection',
          color: '#f59e0b',
          description: 'Some services are connected, others may be experiencing issues'
        };
      case 'disconnected':
        return {
          icon: '🔴',
          text: 'Disconnected',
          color: '#ef4444',
          description: 'No services are connected. Check your internet connection.'
        };
      default:
        return {
          icon: '⚪',
          text: 'Unknown',
          color: '#6b7280',
          description: 'Connection status is unknown'
        };
    }
  };

  const statusInfo = getStatusInfo();

  return (
    <div className="connection-status">
      <div 
        className="status-indicator"
        style={{ color: statusInfo.color }}
        onClick={() => setShowDetails(!showDetails)}
      >
        <span className="status-icon">{statusInfo.icon}</span>
        <span className="status-text">{statusInfo.text}</span>
        {lastUpdate && (
          <span className="last-update">
            Last update: {lastUpdate.toLocaleTimeString()}
          </span>
        )}
      </div>
      
      {showDetails && (
        <div className="status-details">
          <p>{statusInfo.description}</p>
          {status === 'disconnected' && onReconnect && (
            <button onClick={onReconnect} className="reconnect-btn">
              Reconnect
            </button>
          )}
        </div>
      )}
    </div>
  );
}
```

## 📝 Documentation Tasks

### Create API Integration Guide
Create `week1/day5/docs/api-integration.md`:

```markdown
# API Integration Guide

## REST API Best Practices
- Use proper HTTP methods
- Implement error handling
- Add request caching
- Handle rate limiting
- Implement retry logic

## WebSocket Implementation
- Connection management
- Message handling
- Error recovery
- Performance optimization
- Security considerations

## Real-Time Data
- Data synchronization
- Live updates
- Connection status
- Error handling
- Performance monitoring
```

## 🧪 Testing & Validation

### API Testing
- [x] All API endpoints work correctly
- [x] Error handling works
- [x] Caching works properly
- [x] Retry logic functions

### WebSocket Testing
- [x] Connection establishes
- [x] Messages are received
- [x] Reconnection works
- [x] Error handling works

## 📊 Success Criteria

By the end of Day 5, you should have:

✅ **API Integration**: Robust API service layer  
✅ **WebSocket Implementation**: Real-time data connection  
✅ **Error Handling**: Comprehensive error management  
✅ **Performance**: Optimized data fetching  
✅ **Real-Time Updates**: Live data synchronization  

## ✅ Completion Status

All Day 5 deliverables are complete and tested:
- `src/services/ApiService.js` — REST client with TTL cache, retry with backoff, AbortController timeouts
- `src/services/WebSocketService.js` — connect/disconnect, heartbeat, message queue, reconnection
- `src/hooks/useRealTimeData.js`, `useWebSocket.js`, `useApiService.js` — real-time data layer
- `src/components/RealTimeDashboard.jsx`, `ConnectionStatus.jsx`, `ChartContainer.jsx`, `MetricsCard.jsx`
- `src/data/mockData.js` — fallback data, `src/main.jsx` + `index.html` + `vite.config.js` entry/config
- `docs/api-integration.md`

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 5: API & Real-Time Data"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 6**: Review documentation and Agile practices
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [MDN Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [REST API Design](https://restfulapi.net/)
- [WebSocket Best Practices](https://blog.stanko.io/websocket-best-practices-4b1a3b2b8b8b)

---

**Ready for Day 6? Check out [Day 6: Documentation & Agile](../day6/README.md)!** 🚀

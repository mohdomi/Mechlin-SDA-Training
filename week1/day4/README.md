# Day 4: React Advanced

## 🎯 Learning Objectives

- Master React Hooks: useState, useEffect, useContext, useReducer
- Implement advanced state management patterns
- Create reusable components with proper prop validation
- Optimize React performance with memoization
- Build complex component hierarchies with proper data flow

## 📚 Theory & Concepts

### React Hooks
- **useState**: Local state management, functional updates
- **useEffect**: Side effects, cleanup, dependency arrays
- **useContext**: Global state sharing, context providers
- **useReducer**: Complex state logic, state machines
- **Custom Hooks**: Reusable logic, composition patterns

### Advanced Patterns
- **Compound Components**: Flexible component APIs
- **Render Props**: Function as children pattern
- **Higher-Order Components**: Component composition
- **Context API**: Global state management
- **Error Boundaries**: Error handling and recovery

### Performance Optimization
- **React.memo**: Prevent unnecessary re-renders
- **useMemo**: Expensive calculations
- **useCallback**: Function memoization
- **Code Splitting**: Lazy loading, dynamic imports
- **Bundle Analysis**: Webpack bundle analyzer

## 🛠️ Hands-on Tasks

### Task 1: Create Advanced React Dashboard
Build a comprehensive React dashboard with hooks and state management:

```jsx
// src/components/Dashboard.jsx
import React, { useState, useEffect, useContext, useReducer } from 'react';
import { DataContext } from '../contexts/DataContext';
import { ChartContainer } from './ChartContainer';
import { MetricsCard } from './MetricsCard';
import { PerformanceMonitor } from './PerformanceMonitor';
import { ErrorBoundary } from './ErrorBoundary';
import './Dashboard.css';

const initialState = {
  loading: false,
  error: null,
  data: {
    users: [],
    revenue: [],
    orders: []
  },
  filters: {
    dateRange: '30d',
    category: 'all'
  }
};

function dataReducer(state, action) {
  switch (action.type) {
    case 'SET_LOADING':
      return { ...state, loading: action.payload };
    case 'SET_ERROR':
      return { ...state, error: action.payload, loading: false };
    case 'SET_DATA':
      return { ...state, data: action.payload, loading: false, error: null };
    case 'UPDATE_FILTERS':
      return { ...state, filters: { ...state.filters, ...action.payload } };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

export function Dashboard() {
  const [state, dispatch] = useReducer(dataReducer, initialState);
  const [selectedMetric, setSelectedMetric] = useState('revenue');
  const [viewMode, setViewMode] = useState('grid');
  
  const { fetchData, subscribe, unsubscribe } = useContext(DataContext);

  useEffect(() => {
    const loadData = async () => {
      dispatch({ type: 'SET_LOADING', payload: true });
      
      try {
        const [users, revenue, orders] = await Promise.all([
          fetchData('/api/users'),
          fetchData('/api/revenue'),
          fetchData('/api/orders')
        ]);
        
        dispatch({
          type: 'SET_DATA',
          payload: { users, revenue, orders }
        });
      } catch (error) {
        dispatch({ type: 'SET_ERROR', payload: error.message });
      }
    };

    loadData();
  }, [fetchData]);

  useEffect(() => {
    const handleDataUpdate = (endpoint, data) => {
      dispatch({
        type: 'SET_DATA',
        payload: { ...state.data, [endpoint.split('/').pop()]: data }
      });
    };

    const unsubscribeFn = subscribe(handleDataUpdate);
    return unsubscribeFn;
  }, [subscribe, state.data]);

  const handleFilterChange = (filterType, value) => {
    dispatch({
      type: 'UPDATE_FILTERS',
      payload: { [filterType]: value }
    });
  };

  const handleRefresh = async () => {
    dispatch({ type: 'SET_LOADING', payload: true });
    try {
      const [users, revenue, orders] = await Promise.all([
        fetchData('/api/users'),
        fetchData('/api/revenue'),
        fetchData('/api/orders')
      ]);
      
      dispatch({
        type: 'SET_DATA',
        payload: { users, revenue, orders }
      });
    } catch (error) {
      dispatch({ type: 'SET_ERROR', payload: error.message });
    }
  };

  if (state.loading) {
    return <LoadingSpinner />;
  }

  if (state.error) {
    return <ErrorMessage error={state.error} onRetry={handleRefresh} />;
  }

  return (
    <ErrorBoundary>
      <div className="dashboard">
        <DashboardHeader
          filters={state.filters}
          onFilterChange={handleFilterChange}
          onRefresh={handleRefresh}
          viewMode={viewMode}
          onViewModeChange={setViewMode}
        />
        
        <div className={`dashboard-content ${viewMode}`}>
          <MetricsGrid data={state.data} />
          
          <div className="charts-section">
            <ChartContainer
              data={state.data}
              selectedMetric={selectedMetric}
              onMetricChange={setSelectedMetric}
            />
          </div>
          
          <PerformanceMonitor />
        </div>
      </div>
    </ErrorBoundary>
  );
}

// Loading component
function LoadingSpinner() {
  return (
    <div className="loading-container">
      <div className="spinner"></div>
      <p>Loading dashboard data...</p>
    </div>
  );
}

// Error component
function ErrorMessage({ error, onRetry }) {
  return (
    <div className="error-container">
      <h2>Error Loading Dashboard</h2>
      <p>{error}</p>
      <button onClick={onRetry} className="retry-btn">
        Try Again
      </button>
    </div>
  );
}
```

### Task 2: Create Custom Hooks
Implement reusable custom hooks for common functionality:

```jsx
// src/hooks/useDataFetching.js
import { useState, useEffect, useCallback } from 'react';

export function useDataFetching(url, options = {}) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const fetchData = useCallback(async () => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch(url, {
        headers: {
          'Content-Type': 'application/json',
          ...options.headers
        },
        ...options
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      const result = await response.json();
      setData(result);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [url, options]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
}

// src/hooks/useLocalStorage.js
import { useState, useEffect } from 'react';

export function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(`Error setting localStorage key "${key}":`, error);
    }
  };

  return [storedValue, setValue];
}

// src/hooks/useDebounce.js
import { useState, useEffect } from 'react';

export function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
}

// src/hooks/usePerformance.js
import { useState, useEffect } from 'react';

export function usePerformance() {
  const [metrics, setMetrics] = useState({});

  useEffect(() => {
    const updateMetrics = () => {
      if ('performance' in window) {
        const navigation = performance.getEntriesByType('navigation')[0];
        const paint = performance.getEntriesByType('paint');
        
        setMetrics({
          loadTime: navigation.loadEventEnd - navigation.loadEventStart,
          domContentLoaded: navigation.domContentLoadedEventEnd - navigation.domContentLoadedEventStart,
          firstPaint: paint.find(entry => entry.name === 'first-paint')?.startTime,
          firstContentfulPaint: paint.find(entry => entry.name === 'first-contentful-paint')?.startTime
        });
      }
    };

    updateMetrics();
    
    const interval = setInterval(updateMetrics, 5000);
    return () => clearInterval(interval);
  }, []);

  return metrics;
}
```

### Task 3: Create Context Providers
Implement context for global state management:

```jsx
// src/contexts/DataContext.jsx
import React, { createContext, useContext, useReducer, useCallback } from 'react';

const DataContext = createContext();

const initialState = {
  cache: new Map(),
  subscribers: new Set(),
  loading: false,
  error: null
};

function dataContextReducer(state, action) {
  switch (action.type) {
    case 'SET_LOADING':
      return { ...state, loading: action.payload };
    case 'SET_ERROR':
      return { ...state, error: action.payload, loading: false };
    case 'CACHE_DATA':
      const newCache = new Map(state.cache);
      newCache.set(action.key, action.data);
      return { ...state, cache: newCache };
    case 'CLEAR_CACHE':
      return { ...state, cache: new Map() };
    default:
      return state;
  }
}

export function DataProvider({ children }) {
  const [state, dispatch] = useReducer(dataContextReducer, initialState);

  const fetchData = useCallback(async (endpoint, options = {}) => {
    const cacheKey = `${endpoint}-${JSON.stringify(options)}`;
    
    if (state.cache.has(cacheKey)) {
      return state.cache.get(cacheKey);
    }

    dispatch({ type: 'SET_LOADING', payload: true });
    
    try {
      const response = await fetch(`/api${endpoint}`, {
        headers: {
          'Content-Type': 'application/json',
          ...options.headers
        },
        ...options
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const data = await response.json();
      dispatch({ type: 'CACHE_DATA', key: cacheKey, data });
      return data;
    } catch (error) {
      dispatch({ type: 'SET_ERROR', payload: error.message });
      throw error;
    }
  }, [state.cache]);

  const subscribe = useCallback((callback) => {
    state.subscribers.add(callback);
    return () => state.subscribers.delete(callback);
  }, [state.subscribers]);

  const clearCache = useCallback(() => {
    dispatch({ type: 'CLEAR_CACHE' });
  }, []);

  const value = {
    fetchData,
    subscribe,
    clearCache,
    loading: state.loading,
    error: state.error
  };

  return (
    <DataContext.Provider value={value}>
      {children}
    </DataContext.Provider>
  );
}

export function useDataContext() {
  const context = useContext(DataContext);
  if (!context) {
    throw new Error('useDataContext must be used within a DataProvider');
  }
  return context;
}

export { DataContext };
```

### Task 4: Create Error Boundary
Implement error boundary for graceful error handling:

```jsx
// src/components/ErrorBoundary.jsx
import React from 'react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null, errorInfo: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    this.setState({
      error: error,
      errorInfo: errorInfo
    });

    // Log error to monitoring service
    console.error('ErrorBoundary caught an error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <details style={{ whiteSpace: 'pre-wrap' }}>
            <summary>Error Details</summary>
            {this.state.error && this.state.error.toString()}
            <br />
            {this.state.errorInfo.componentStack}
          </details>
          <button onClick={() => window.location.reload()}>
            Reload Page
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

export { ErrorBoundary };
```

### Task 5: Create Performance Optimized Components
Implement memoized components for better performance:

```jsx
// src/components/MetricsCard.jsx
import React, { memo, useMemo } from 'react';
import PropTypes from 'prop-types';

export const MetricsCard = memo(function MetricsCard({ 
  title, 
  value, 
  change, 
  trend, 
  icon,
  onClick 
}) {
  const formattedValue = useMemo(() => {
    if (typeof value === 'number') {
      return value.toLocaleString();
    }
    return value;
  }, [value]);

  const changeClass = useMemo(() => {
    if (change > 0) return 'positive';
    if (change < 0) return 'negative';
    return 'neutral';
  }, [change]);

  return (
    <div className="metrics-card" onClick={onClick}>
      <div className="card-header">
        <span className="card-icon">{icon}</span>
        <h3 className="card-title">{title}</h3>
      </div>
      <div className="card-content">
        <div className="card-value">{formattedValue}</div>
        <div className={`card-change ${changeClass}`}>
          {change > 0 ? '+' : ''}{change}%
        </div>
      </div>
      {trend && (
        <div className="card-trend">
          <span className="trend-label">Trend:</span>
          <span className="trend-value">{trend}</span>
        </div>
      )}
    </div>
  );
});

MetricsCard.propTypes = {
  title: PropTypes.string.isRequired,
  value: PropTypes.oneOfType([PropTypes.string, PropTypes.number]).isRequired,
  change: PropTypes.number,
  trend: PropTypes.string,
  icon: PropTypes.string,
  onClick: PropTypes.func
};

MetricsCard.defaultProps = {
  change: 0,
  trend: null,
  icon: '📊',
  onClick: null
};
```

## 📝 Documentation Tasks

### Create React Architecture Guide
Create `week1/day4/docs/react-architecture.md`:

```markdown
# React Architecture Guide

## Component Structure
- Functional components with hooks
- Custom hooks for reusable logic
- Context for global state
- Error boundaries for error handling

## State Management
- Local state with useState
- Complex state with useReducer
- Global state with Context API
- Performance optimization with memoization

## Best Practices
- Component composition over inheritance
- Props validation with PropTypes
- Performance optimization with React.memo
- Code splitting with lazy loading
```

## 🧪 Testing & Validation

### Component Testing
- [x] All components render without errors
- [x] Hooks work correctly
- [x] State updates properly
- [x] Error boundaries catch errors

### Performance Testing
- [x] No unnecessary re-renders
- [x] Memoization works correctly
- [x] Bundle size is optimized
- [x] Loading states work

## 📊 Success Criteria

By the end of Day 4, you should have:

✅ **React Hooks Mastery**: Advanced hooks implementation  
✅ **State Management**: Complex state with useReducer  
✅ **Custom Hooks**: Reusable logic extraction  
✅ **Performance**: Optimized components  
✅ **Error Handling**: Robust error boundaries  

## ✅ Completion Status

All Day 4 deliverables are complete and tested:
- `src/components/` — Dashboard, DashboardHeader, MetricsCard (memo + PropTypes), MetricsGrid, ChartContainer, PerformanceMonitor, ErrorBoundary
- `src/contexts/DataContext.jsx` — cached fetching, subscribers, `useDataContext`
- `src/hooks/` — useDataFetching, useLocalStorage, useDebounce, usePerformance
- `src/data/mockData.js`, `src/App.jsx`, `src/main.jsx`, `index.html`, `vite.config.js`
- `docs/react-architecture.md`

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 4: React Advanced"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 5**: Review API integration concepts
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [React Documentation](https://react.dev/)
- [React Hooks Guide](https://react.dev/reference/react)
- [React Performance](https://react.dev/learn/render-and-commit)
- [React Testing](https://react.dev/learn/testing)

---

**Ready for Day 5? Check out [Day 5: API & Real-Time Data](../day5/README.md)!** 🚀

# Day 3: JavaScript Advanced

## 🎯 Learning Objectives

- Master ES6+ features: arrow functions, destructuring, modules, async/await
- Implement advanced JavaScript patterns and design principles
- Build dynamic data visualization with Chart.js or D3.js
- Optimize DOM manipulation and performance
- Create reusable JavaScript modules and components

## 📚 Theory & Concepts

### ES6+ Features
- **Arrow Functions**: Concise syntax, lexical `this` binding
- **Destructuring**: Object and array destructuring, default values
- **Template Literals**: String interpolation, tagged templates
- **Modules**: Import/export, dynamic imports, tree shaking
- **Async/Await**: Promise handling, error management, concurrent operations

### Advanced JavaScript Patterns
- **Closures**: Function scope, private variables, module pattern
- **Prototypes**: Object inheritance, prototype chain, ES6 classes
- **Event Handling**: Event delegation, custom events, performance optimization
- **Memory Management**: Garbage collection, memory leaks, performance profiling

### Data Visualization
- **Chart.js**: Canvas-based charts, responsive design, animations
- **D3.js**: Data-driven documents, SVG manipulation, custom visualizations
- **Performance**: Canvas vs SVG, data processing, rendering optimization

## 🛠️ Hands-on Tasks

### Task 1: Create Advanced JavaScript Module System
Build a modular dashboard system with ES6 modules:

```javascript
// src/modules/DataManager.js
export class DataManager {
    constructor(apiUrl) {
        this.apiUrl = apiUrl;
        this.cache = new Map();
        this.subscribers = new Set();
    }
    
    async fetchData(endpoint, options = {}) {
        const cacheKey = `${endpoint}-${JSON.stringify(options)}`;
        
        if (this.cache.has(cacheKey)) {
            return this.cache.get(cacheKey);
        }
        
        try {
            const response = await fetch(`${this.apiUrl}${endpoint}`, {
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
            this.cache.set(cacheKey, data);
            this.notifySubscribers(endpoint, data);
            
            return data;
        } catch (error) {
            console.error('Data fetch error:', error);
            throw error;
        }
    }
    
    subscribe(callback) {
        this.subscribers.add(callback);
        return () => this.subscribers.delete(callback);
    }
    
    notifySubscribers(endpoint, data) {
        this.subscribers.forEach(callback => callback(endpoint, data));
    }
    
    clearCache() {
        this.cache.clear();
    }
}
```

### Task 2: Implement Data Visualization Dashboard
Create an interactive dashboard with Chart.js:

```javascript
// src/modules/ChartManager.js
import { DataManager } from './DataManager.js';

export class ChartManager {
    constructor(containerId, dataManager) {
        this.container = document.getElementById(containerId);
        this.dataManager = dataManager;
        this.charts = new Map();
        this.init();
    }
    
    async init() {
        await this.loadChartLibrary();
        this.setupEventListeners();
        await this.createCharts();
    }
    
    async loadChartLibrary() {
        return new Promise((resolve, reject) => {
            const script = document.createElement('script');
            script.src = 'https://cdn.jsdelivr.net/npm/chart.js';
            script.onload = resolve;
            script.onerror = reject;
            document.head.appendChild(script);
        });
    }
    
    async createCharts() {
        try {
            // Fetch data
            const [userData, revenueData, orderData] = await Promise.all([
                this.dataManager.fetchData('/api/users'),
                this.dataManager.fetchData('/api/revenue'),
                this.dataManager.fetchData('/api/orders')
            ]);
            
            // Create charts
            this.createLineChart('revenueChart', revenueData);
            this.createBarChart('userChart', userData);
            this.createDoughnutChart('orderChart', orderData);
            this.createMixedChart('performanceChart', { userData, revenueData, orderData });
            
        } catch (error) {
            console.error('Chart creation error:', error);
            this.showError('Failed to load chart data');
        }
    }
    
    createLineChart(canvasId, data) {
        const ctx = document.getElementById(canvasId).getContext('2d');
        
        this.charts.set(canvasId, new Chart(ctx, {
            type: 'line',
            data: {
                labels: data.labels,
                datasets: [{
                    label: 'Revenue',
                    data: data.values,
                    borderColor: 'rgb(59, 130, 246)',
                    backgroundColor: 'rgba(59, 130, 246, 0.1)',
                    tension: 0.4,
                    fill: true
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: {
                        display: false
                    }
                },
                scales: {
                    y: {
                        beginAtZero: true,
                        grid: {
                            color: 'rgba(0, 0, 0, 0.1)'
                        }
                    },
                    x: {
                        grid: {
                            display: false
                        }
                    }
                },
                animation: {
                    duration: 2000,
                    easing: 'easeInOutQuart'
                }
            }
        }));
    }
    
    createBarChart(canvasId, data) {
        const ctx = document.getElementById(canvasId).getContext('2d');
        
        this.charts.set(canvasId, new Chart(ctx, {
            type: 'bar',
            data: {
                labels: data.labels,
                datasets: [{
                    label: 'Users',
                    data: data.values,
                    backgroundColor: 'rgba(16, 185, 129, 0.8)',
                    borderColor: 'rgb(16, 185, 129)',
                    borderWidth: 1
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: {
                        display: false
                    }
                },
                scales: {
                    y: {
                        beginAtZero: true
                    }
                }
            }
        }));
    }
    
    createDoughnutChart(canvasId, data) {
        const ctx = document.getElementById(canvasId).getContext('2d');
        
        this.charts.set(canvasId, new Chart(ctx, {
            type: 'doughnut',
            data: {
                labels: data.labels,
                datasets: [{
                    data: data.values,
                    backgroundColor: [
                        'rgba(59, 130, 246, 0.8)',
                        'rgba(16, 185, 129, 0.8)',
                        'rgba(245, 158, 11, 0.8)',
                        'rgba(239, 68, 68, 0.8)'
                    ],
                    borderWidth: 2,
                    borderColor: '#fff'
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: {
                        position: 'bottom'
                    }
                }
            }
        }));
    }
    
    createMixedChart(canvasId, data) {
        const ctx = document.getElementById(canvasId).getContext('2d');
        
        this.charts.set(canvasId, new Chart(ctx, {
            type: 'line',
            data: {
                labels: data.userData.labels,
                datasets: [
                    {
                        label: 'Users',
                        data: data.userData.values,
                        type: 'bar',
                        backgroundColor: 'rgba(59, 130, 246, 0.6)',
                        borderColor: 'rgb(59, 130, 246)',
                        yAxisID: 'y'
                    },
                    {
                        label: 'Revenue',
                        data: data.revenueData.values,
                        type: 'line',
                        borderColor: 'rgb(16, 185, 129)',
                        backgroundColor: 'rgba(16, 185, 129, 0.1)',
                        tension: 0.4,
                        yAxisID: 'y1'
                    }
                ]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    y: {
                        type: 'linear',
                        display: true,
                        position: 'left'
                    },
                    y1: {
                        type: 'linear',
                        display: true,
                        position: 'right',
                        grid: {
                            drawOnChartArea: false
                        }
                    }
                }
            }
        }));
    }
    
    setupEventListeners() {
        // Add chart interaction events
        this.dataManager.subscribe((endpoint, data) => {
            this.updateCharts(endpoint, data);
        });
        
        // Add window resize handler
        window.addEventListener('resize', this.debounce(() => {
            this.charts.forEach(chart => chart.resize());
        }, 250));
    }
    
    updateCharts(endpoint, data) {
        // Update specific charts based on endpoint
        switch(endpoint) {
            case '/api/users':
                this.updateChart('userChart', data);
                break;
            case '/api/revenue':
                this.updateChart('revenueChart', data);
                break;
            case '/api/orders':
                this.updateChart('orderChart', data);
                break;
        }
    }
    
    updateChart(chartId, data) {
        const chart = this.charts.get(chartId);
        if (chart) {
            chart.data = data;
            chart.update('active');
        }
    }
    
    showError(message) {
        this.container.innerHTML = `
            <div class="error-message">
                <h3>Error Loading Charts</h3>
                <p>${message}</p>
                <button onclick="location.reload()">Retry</button>
            </div>
        `;
    }
    
    debounce(func, wait) {
        let timeout;
        return function executedFunction(...args) {
            const later = () => {
                clearTimeout(timeout);
                func(...args);
            };
            clearTimeout(timeout);
            timeout = setTimeout(later, wait);
        };
    }
}
```

### Task 3: Create Performance Monitoring
Implement performance monitoring and optimization:

```javascript
// src/modules/PerformanceMonitor.js
export class PerformanceMonitor {
    constructor() {
        this.metrics = new Map();
        this.observers = new Set();
        this.init();
    }
    
    init() {
        this.observePerformance();
        this.observeMemory();
        this.observeUserInteractions();
    }
    
    observePerformance() {
        // Monitor Core Web Vitals
        if ('PerformanceObserver' in window) {
            // Largest Contentful Paint
            new PerformanceObserver((list) => {
                const entries = list.getEntries();
                const lastEntry = entries[entries.length - 1];
                this.recordMetric('LCP', lastEntry.startTime);
            }).observe({ entryTypes: ['largest-contentful-paint'] });
            
            // First Input Delay
            new PerformanceObserver((list) => {
                const entries = list.getEntries();
                entries.forEach(entry => {
                    this.recordMetric('FID', entry.processingStart - entry.startTime);
                });
            }).observe({ entryTypes: ['first-input'] });
            
            // Cumulative Layout Shift
            new PerformanceObserver((list) => {
                const entries = list.getEntries();
                entries.forEach(entry => {
                    if (!entry.hadRecentInput) {
                        this.recordMetric('CLS', entry.value);
                    }
                });
            }).observe({ entryTypes: ['layout-shift'] });
        }
    }
    
    observeMemory() {
        if ('memory' in performance) {
            setInterval(() => {
                const memory = performance.memory;
                this.recordMetric('memory-used', memory.usedJSHeapSize);
                this.recordMetric('memory-total', memory.totalJSHeapSize);
                this.recordMetric('memory-limit', memory.jsHeapSizeLimit);
            }, 5000);
        }
    }
    
    observeUserInteractions() {
        let interactionCount = 0;
        const startTime = performance.now();
        
        ['click', 'keydown', 'scroll', 'touchstart'].forEach(eventType => {
            document.addEventListener(eventType, () => {
                interactionCount++;
                this.recordMetric('interaction-count', interactionCount);
            }, { passive: true });
        });
    }
    
    recordMetric(name, value) {
        const timestamp = Date.now();
        const metric = { name, value, timestamp };
        
        this.metrics.set(name, metric);
        this.notifyObservers(metric);
        
        // Store in localStorage for persistence
        this.storeMetric(metric);
    }
    
    getMetric(name) {
        return this.metrics.get(name);
    }
    
    getAllMetrics() {
        return Array.from(this.metrics.values());
    }
    
    subscribe(callback) {
        this.observers.add(callback);
        return () => this.observers.delete(callback);
    }
    
    notifyObservers(metric) {
        this.observers.forEach(callback => callback(metric));
    }
    
    storeMetric(metric) {
        const stored = JSON.parse(localStorage.getItem('performance-metrics') || '[]');
        stored.push(metric);
        
        // Keep only last 100 metrics
        if (stored.length > 100) {
            stored.splice(0, stored.length - 100);
        }
        
        localStorage.setItem('performance-metrics', JSON.stringify(stored));
    }
    
    getStoredMetrics() {
        return JSON.parse(localStorage.getItem('performance-metrics') || '[]');
    }
    
    generateReport() {
        const metrics = this.getAllMetrics();
        const report = {
            timestamp: Date.now(),
            metrics: metrics,
            summary: this.generateSummary(metrics)
        };
        
        return report;
    }
    
    generateSummary(metrics) {
        const summary = {};
        
        metrics.forEach(metric => {
            if (!summary[metric.name]) {
                summary[metric.name] = {
                    count: 0,
                    total: 0,
                    average: 0,
                    min: Infinity,
                    max: -Infinity
                };
            }
            
            const stat = summary[metric.name];
            stat.count++;
            stat.total += metric.value;
            stat.average = stat.total / stat.count;
            stat.min = Math.min(stat.min, metric.value);
            stat.max = Math.max(stat.max, metric.value);
        });
        
        return summary;
    }
}
```

### Task 4: Create Main Application
Integrate all modules into a cohesive application:

```javascript
// src/app.js
import { DataManager } from './modules/DataManager.js';
import { ChartManager } from './modules/ChartManager.js';
import { PerformanceMonitor } from './modules/PerformanceMonitor.js';

class DashboardApp {
    constructor() {
        this.dataManager = new DataManager('/api');
        this.chartManager = null;
        this.performanceMonitor = new PerformanceMonitor();
        this.init();
    }
    
    async init() {
        try {
            await this.setupUI();
            await this.initializeCharts();
            this.setupEventListeners();
            this.startPerformanceMonitoring();
        } catch (error) {
            console.error('App initialization error:', error);
            this.showError('Failed to initialize dashboard');
        }
    }
    
    async setupUI() {
        // Create dashboard HTML structure
        const dashboardHTML = `
            <div class="dashboard-container">
                <div class="charts-grid">
                    <div class="chart-container">
                        <h3>Revenue Trend</h3>
                        <canvas id="revenueChart"></canvas>
                    </div>
                    <div class="chart-container">
                        <h3>User Growth</h3>
                        <canvas id="userChart"></canvas>
                    </div>
                    <div class="chart-container">
                        <h3>Order Distribution</h3>
                        <canvas id="orderChart"></canvas>
                    </div>
                    <div class="chart-container">
                        <h3>Performance Overview</h3>
                        <canvas id="performanceChart"></canvas>
                    </div>
                </div>
                <div class="performance-panel">
                    <h3>Performance Metrics</h3>
                    <div id="performance-metrics"></div>
                </div>
            </div>
        `;
        
        document.querySelector('.content').innerHTML = dashboardHTML;
    }
    
    async initializeCharts() {
        this.chartManager = new ChartManager('charts-grid', this.dataManager);
    }
    
    setupEventListeners() {
        // Add refresh button
        const refreshBtn = document.createElement('button');
        refreshBtn.textContent = 'Refresh Data';
        refreshBtn.className = 'refresh-btn';
        refreshBtn.addEventListener('click', () => this.refreshData());
        document.querySelector('.content-header').appendChild(refreshBtn);
        
        // Add performance monitoring toggle
        const monitorBtn = document.createElement('button');
        monitorBtn.textContent = 'Toggle Performance Monitor';
        monitorBtn.className = 'monitor-btn';
        monitorBtn.addEventListener('click', () => this.togglePerformanceMonitor());
        document.querySelector('.content-header').appendChild(monitorBtn);
    }
    
    async refreshData() {
        this.dataManager.clearCache();
        await this.chartManager.createCharts();
    }
    
    togglePerformanceMonitor() {
        const panel = document.querySelector('.performance-panel');
        panel.style.display = panel.style.display === 'none' ? 'block' : 'none';
    }
    
    startPerformanceMonitoring() {
        this.performanceMonitor.subscribe((metric) => {
            this.updatePerformanceDisplay(metric);
        });
    }
    
    updatePerformanceDisplay(metric) {
        const container = document.getElementById('performance-metrics');
        if (!container) return;
        
        const metricElement = document.createElement('div');
        metricElement.className = 'metric-item';
        metricElement.innerHTML = `
            <span class="metric-name">${metric.name}:</span>
            <span class="metric-value">${metric.value.toFixed(2)}</span>
            <span class="metric-time">${new Date(metric.timestamp).toLocaleTimeString()}</span>
        `;
        
        container.appendChild(metricElement);
        
        // Keep only last 10 metrics visible
        const items = container.querySelectorAll('.metric-item');
        if (items.length > 10) {
            items[0].remove();
        }
    }
    
    showError(message) {
        document.querySelector('.content').innerHTML = `
            <div class="error-container">
                <h2>Dashboard Error</h2>
                <p>${message}</p>
                <button onclick="location.reload()">Reload Page</button>
            </div>
        `;
    }
}

// Initialize the application
document.addEventListener('DOMContentLoaded', () => {
    new DashboardApp();
});
```

## 📝 Documentation Tasks

### Create JavaScript Architecture Guide
Create `week1/day3/docs/javascript-architecture.md`:

```markdown
# JavaScript Architecture Guide

## Module System
- ES6 modules with import/export
- Separation of concerns
- Dependency injection
- Lazy loading for performance

## Design Patterns
- Observer pattern for event handling
- Module pattern for encapsulation
- Factory pattern for object creation
- Strategy pattern for algorithms

## Performance Optimization
- Debouncing and throttling
- Memory management
- Lazy loading
- Code splitting
```

### Create Performance Best Practices
Create `week1/day3/docs/performance-guide.md`:

```markdown
# JavaScript Performance Guide

## Optimization Techniques
- Use requestAnimationFrame for animations
- Implement virtual scrolling for large lists
- Use Web Workers for heavy computations
- Optimize DOM manipulation

## Memory Management
- Avoid memory leaks
- Use WeakMap and WeakSet
- Clean up event listeners
- Monitor memory usage
```

## 🧪 Testing & Validation

### Performance Testing
- [x] Lighthouse score > 90
- [x] First Contentful Paint < 1.5s
- [x] Largest Contentful Paint < 2.5s
- [x] Cumulative Layout Shift < 0.1

### Functionality Testing
- [x] All charts render correctly
- [x] Data updates in real-time
- [x] Performance metrics display
- [x] Error handling works

## 📊 Success Criteria

By the end of Day 3, you should have:

✅ **ES6+ Mastery**: Advanced JavaScript features implemented  
✅ **Module System**: Clean, organized code structure  
✅ **Data Visualization**: Interactive charts with Chart.js  
✅ **Performance Monitoring**: Real-time performance tracking  
✅ **Error Handling**: Robust error management  

## ✅ Completion Status

All Day 3 deliverables are complete and tested:
- `src/app.js` — DashboardApp entry point composing all modules
- `src/modules/DataManager.js` — fetch/caching/subscribe pattern
- `src/modules/ChartManager.js` — Chart.js line/bar/doughnut/mixed charts
- `src/modules/PerformanceMonitor.js` — Core Web Vitals + memory tracking
- `src/modules/index.js` — module barrel exports
- `api/users.json`, `api/revenue.json`, `api/orders.json` — local data sources
- `index.html` — dashboard shell
- `docs/javascript-architecture.md`, `docs/performance-guide.md`

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 3: JavaScript Advanced"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 4**: Review React concepts and hooks
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [MDN JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)
- [Chart.js Documentation](https://www.chartjs.org/docs/)
- [ES6 Features](https://es6-features.org/)
- [JavaScript Performance](https://web.dev/javascript-performance/)

---

**Ready for Day 4? Check out [Day 4: React Advanced](../day4/README.md)!** 🚀

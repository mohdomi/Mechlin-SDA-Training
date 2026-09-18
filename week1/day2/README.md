# Day 2: HTML5 & Advanced CSS

## 🎯 Learning Objectives

- Master HTML5 semantic elements and accessibility
- Implement advanced CSS layouts with Flexbox and Grid
- Create smooth animations and transitions
- Build responsive designs with modern CSS
- Integrate CSS frameworks (Tailwind CSS or Material UI)

## 📚 Theory & Concepts

### HTML5 Semantic Elements
- **Semantic HTML**: `<header>`, `<nav>`, `<main>`, `<section>`, `<article>`, `<aside>`, `<footer>`
- **Accessibility**: ARIA attributes, semantic roles, keyboard navigation
- **SEO Optimization**: Proper heading hierarchy, meta tags, structured data

### Advanced CSS Layouts
- **Flexbox**: One-dimensional layouts, alignment, distribution
- **CSS Grid**: Two-dimensional layouts, grid areas, responsive grids
- **CSS Custom Properties**: Variables, theming, dynamic values
- **CSS Modules**: Scoped styles, component-based CSS

### Modern CSS Features
- **Animations**: Keyframes, transitions, transforms
- **Responsive Design**: Media queries, container queries, fluid typography
- **CSS Frameworks**: Tailwind CSS, Material UI, Bootstrap

## 🛠️ Hands-on Tasks

### Task 1: Create Responsive Dashboard Layout
Build a multi-page responsive dashboard with the following structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Advanced Dashboard</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <header class="header">
        <nav class="nav">
            <div class="nav-brand">
                <h1>Dashboard</h1>
            </div>
            <ul class="nav-menu">
                <li><a href="#home">Home</a></li>
                <li><a href="#analytics">Analytics</a></li>
                <li><a href="#settings">Settings</a></li>
            </ul>
        </nav>
    </header>
    
    <main class="main">
        <aside class="sidebar">
            <nav class="sidebar-nav">
                <ul>
                    <li><a href="#dashboard">Dashboard</a></li>
                    <li><a href="#users">Users</a></li>
                    <li><a href="#reports">Reports</a></li>
                </ul>
            </nav>
        </aside>
        
        <section class="content">
            <div class="dashboard-grid">
                <div class="card">
                    <h3>Total Users</h3>
                    <p class="metric">1,234</p>
                </div>
                <div class="card">
                    <h3>Revenue</h3>
                    <p class="metric">$45,678</p>
                </div>
                <div class="card">
                    <h3>Orders</h3>
                    <p class="metric">567</p>
                </div>
            </div>
        </section>
    </main>
    
    <footer class="footer">
        <p>&copy; 2024 Advanced Dashboard. All rights reserved.</p>
    </footer>
</body>
</html>
```

### Task 2: Implement Advanced CSS
Create `styles.css` with modern CSS features:

```css
/* CSS Custom Properties */
:root {
    --primary-color: #3b82f6;
    --secondary-color: #64748b;
    --success-color: #10b981;
    --warning-color: #f59e0b;
    --error-color: #ef4444;
    --background-color: #f8fafc;
    --text-color: #1e293b;
    --border-radius: 0.5rem;
    --shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1);
    --transition: all 0.3s ease;
}

/* Reset and Base Styles */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
    line-height: 1.6;
    color: var(--text-color);
    background-color: var(--background-color);
}

/* Header Styles */
.header {
    background: white;
    box-shadow: var(--shadow);
    position: sticky;
    top: 0;
    z-index: 100;
}

.nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem 2rem;
    max-width: 1200px;
    margin: 0 auto;
}

.nav-brand h1 {
    color: var(--primary-color);
    font-size: 1.5rem;
    font-weight: 700;
}

.nav-menu {
    display: flex;
    list-style: none;
    gap: 2rem;
}

.nav-menu a {
    text-decoration: none;
    color: var(--text-color);
    font-weight: 500;
    transition: var(--transition);
    position: relative;
}

.nav-menu a:hover {
    color: var(--primary-color);
}

.nav-menu a::after {
    content: '';
    position: absolute;
    bottom: -5px;
    left: 0;
    width: 0;
    height: 2px;
    background: var(--primary-color);
    transition: var(--transition);
}

.nav-menu a:hover::after {
    width: 100%;
}

/* Main Layout */
.main {
    display: grid;
    grid-template-columns: 250px 1fr;
    min-height: calc(100vh - 80px);
    max-width: 1200px;
    margin: 0 auto;
    gap: 2rem;
    padding: 2rem;
}

/* Sidebar */
.sidebar {
    background: white;
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
    padding: 1.5rem;
    height: fit-content;
}

.sidebar-nav ul {
    list-style: none;
}

.sidebar-nav li {
    margin-bottom: 0.5rem;
}

.sidebar-nav a {
    display: block;
    padding: 0.75rem 1rem;
    text-decoration: none;
    color: var(--text-color);
    border-radius: var(--border-radius);
    transition: var(--transition);
}

.sidebar-nav a:hover {
    background: var(--primary-color);
    color: white;
}

/* Content Area */
.content {
    background: white;
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
    padding: 2rem;
}

/* Dashboard Grid */
.dashboard-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 1.5rem;
    margin-bottom: 2rem;
}

.card {
    background: linear-gradient(135deg, var(--primary-color), #1d4ed8);
    color: white;
    padding: 2rem;
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
    transition: var(--transition);
    position: relative;
    overflow: hidden;
}

.card::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: linear-gradient(45deg, transparent, rgba(255, 255, 255, 0.1));
    transform: translateX(-100%);
    transition: var(--transition);
}

.card:hover::before {
    transform: translateX(100%);
}

.card:hover {
    transform: translateY(-5px);
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
}

.card h3 {
    font-size: 0.875rem;
    font-weight: 500;
    opacity: 0.9;
    margin-bottom: 0.5rem;
}

.metric {
    font-size: 2rem;
    font-weight: 700;
    margin: 0;
}

/* Footer */
.footer {
    background: var(--text-color);
    color: white;
    text-align: center;
    padding: 1rem;
    margin-top: 2rem;
}

/* Responsive Design */
@media (max-width: 768px) {
    .main {
        grid-template-columns: 1fr;
        padding: 1rem;
    }
    
    .nav {
        flex-direction: column;
        gap: 1rem;
    }
    
    .nav-menu {
        flex-direction: column;
        gap: 1rem;
    }
    
    .dashboard-grid {
        grid-template-columns: 1fr;
    }
}

/* Animations */
@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(30px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.card {
    animation: fadeInUp 0.6s ease-out;
}

.card:nth-child(2) {
    animation-delay: 0.1s;
}

.card:nth-child(3) {
    animation-delay: 0.2s;
}
```

### Task 3: Add Interactive Features
Enhance the dashboard with JavaScript interactions:

```javascript
// Add to script.js
document.addEventListener('DOMContentLoaded', function() {
    // Smooth scrolling for navigation
    const navLinks = document.querySelectorAll('.nav-menu a, .sidebar-nav a');
    
    navLinks.forEach(link => {
        link.addEventListener('click', function(e) {
            e.preventDefault();
            const targetId = this.getAttribute('href').substring(1);
            const targetElement = document.getElementById(targetId);
            
            if (targetElement) {
                targetElement.scrollIntoView({
                    behavior: 'smooth',
                    block: 'start'
                });
            }
        });
    });
    
    // Add loading animation to cards
    const cards = document.querySelectorAll('.card');
    cards.forEach((card, index) => {
        card.style.opacity = '0';
        card.style.transform = 'translateY(20px)';
        
        setTimeout(() => {
            card.style.transition = 'all 0.6s ease';
            card.style.opacity = '1';
            card.style.transform = 'translateY(0)';
        }, index * 100);
    });
    
    // Add hover effects to navigation
    const sidebarLinks = document.querySelectorAll('.sidebar-nav a');
    sidebarLinks.forEach(link => {
        link.addEventListener('mouseenter', function() {
            this.style.transform = 'translateX(10px)';
        });
        
        link.addEventListener('mouseleave', function() {
            this.style.transform = 'translateX(0)';
        });
    });
});
```

### Task 4: Implement Tailwind CSS (Alternative)
If you prefer using Tailwind CSS:

```bash
# Install Tailwind CSS
npm install -D tailwindcss
npx tailwindcss init
```

Create `tailwind.config.js`:
```javascript
module.exports = {
  content: ["./src/**/*.{html,js}"],
  theme: {
    extend: {
      colors: {
        primary: '#3b82f6',
        secondary: '#64748b',
      },
      animation: {
        'fade-in-up': 'fadeInUp 0.6s ease-out',
      }
    },
  },
  plugins: [],
}
```

## 📝 Documentation Tasks

### Create CSS Architecture Guide
Create `week1/day2/docs/css-architecture.md`:

```markdown
# CSS Architecture Guide

## Methodology
We use a component-based CSS architecture with the following principles:

### 1. CSS Custom Properties
- Define design tokens as CSS variables
- Enable theming and dynamic values
- Maintain consistency across components

### 2. Component-Based Structure
- Each component has its own CSS
- Use BEM methodology for naming
- Scoped styles prevent conflicts

### 3. Responsive Design
- Mobile-first approach
- Flexible grid systems
- Fluid typography and spacing

### 4. Performance Optimization
- Minimal CSS footprint
- Efficient selectors
- Critical CSS inlining
```

### Create Responsive Design Checklist
Create `week1/day2/docs/responsive-checklist.md`:

```markdown
# Responsive Design Checklist

## Layout
- [ ] Flexible grid system implemented
- [ ] Breakpoints defined for all screen sizes
- [ ] Navigation adapts to mobile devices
- [ ] Images are responsive

## Typography
- [ ] Font sizes scale appropriately
- [ ] Line height is readable on all devices
- [ ] Text is not too small on mobile

## Performance
- [ ] CSS is optimized and minified
- [ ] Critical CSS is inlined
- [ ] Unused CSS is removed
- [ ] Images are optimized

## Accessibility
- [ ] Color contrast meets WCAG standards
- [ ] Focus states are visible
- [ ] Keyboard navigation works
- [ ] Screen reader friendly
```

## 🧪 Testing & Validation

### Cross-Browser Testing
Test your dashboard in:
- [x] Chrome (latest)
- [x] Firefox (latest)
- [x] Safari (latest)
- [x] Edge (latest)

### Responsive Testing
Test at different screen sizes:
- [x] Mobile (320px - 768px)
- [x] Tablet (768px - 1024px)
- [x] Desktop (1024px+)

### Performance Testing
- [x] Lighthouse score > 90
- [x] CSS bundle size < 50KB
- [x] No layout shifts
- [x] Smooth animations (60fps)

## 📊 Success Criteria

By the end of Day 2, you should have:

✅ **Responsive Layout**: Dashboard works on all screen sizes  
✅ **Modern CSS**: Flexbox, Grid, and animations implemented  
✅ **Accessibility**: Semantic HTML and ARIA attributes  
✅ **Performance**: Optimized CSS and smooth animations  
✅ **Documentation**: Clear architecture and responsive guidelines  

## ✅ Completion Status

All Day 2 deliverables are complete and tested:
- `code/index.html` — semantic dashboard (header/nav/sidebar/cards/footer)
- `code/styles.css` — custom properties, Flexbox/Grid, breakpoints at 1024/768/480px, fadeInUp/slideInLeft animations, focus + print styles
- `code/script.js` — navigation, staggered card animations, sidebar interactions, scroll observer, mobile menu, keyboard shortcuts
- `docs/css-architecture.md`, `docs/responsive-checklist.md` — completed (image items N/A, no raster images)
- Verified: cross-browser, all three screen ranges, smooth animations, no layout shifts

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 2: HTML5 & Advanced CSS"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 3**: Review JavaScript ES6+ concepts
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [MDN CSS Documentation](https://developer.mozilla.org/en-US/docs/Web/CSS)
- [CSS Grid Guide](https://css-tricks.com/snippets/css/complete-guide-grid/)
- [Flexbox Guide](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)

---

**Ready for Day 3? Check out [Day 3: JavaScript Advanced](../day3/README.md)!** 🚀

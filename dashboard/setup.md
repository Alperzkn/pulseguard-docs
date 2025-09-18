# PulseGuard Dashboard Setup Guide

## Prerequisites

Before setting up the PulseGuard Dashboard, ensure you have:

1. **Node.js 18+** - Required for running the React application
2. **PulseGuard API** - Running at your server endpoint
3. **API Key** - Valid API key for accessing PulseGuard API

## Environment Setup

### 1. Clone and Install

```bash
# Clone the repository
git clone your-repo-url pulseguard-dashboard
cd pulseguard-dashboard

# Install dependencies
npm install
```

### 2. Environment Configuration

Create environment files for different deployment stages:

#### Development (.env.development)
```bash
VITE_API_BASE_URL=http://localhost:3000/api/v1
VITE_API_KEY=your-development-api-key
VITE_WS_URL=ws://localhost:3000/ws
VITE_ENVIRONMENT=development
VITE_DEBUG=true
```

#### Production (.env.production)
```bash
VITE_API_BASE_URL=http://your-droplet-ip:3000/api/v1
VITE_API_KEY=your-production-api-key
VITE_WS_URL=ws://your-droplet-ip:3000/ws
VITE_ENVIRONMENT=production
VITE_DEBUG=false
```

#### Local (.env)
```bash
VITE_API_BASE_URL=http://164.92.238.238:3000/api/v1
VITE_API_KEY=your-api-key
VITE_WS_URL=ws://164.92.238.238:3000/ws
VITE_ENVIRONMENT=development
```

### 3. API Connectivity Test

Before starting the dashboard, verify API connectivity:

```bash
# Test API connection
curl -H "X-API-Key: your-api-key" http://your-api-url/api/v1/system/health

# Should return:
{
  "success": true,
  "data": {
    "status": "healthy",
    "timestamp": "2025-09-18T10:30:00Z"
  }
}
```

## Development

### Local Development Server

```bash
# Start development server with hot reload
npm run dev

# Dashboard will be available at:
# http://localhost:3001
```

### Development Features

- **Hot Module Replacement** - Instant updates during development
- **TypeScript Support** - Full type checking and IntelliSense
- **ESLint Integration** - Code quality and consistency
- **Tailwind CSS** - Utility-first styling with live recompilation

### Development Scripts

```bash
# Start development server
npm run dev

# Type checking
npm run lint

# Build for production
npm run build

# Preview production build
npm run preview
```

## Production Build

### Build Process

```bash
# Create optimized production build
npm run build

# Files will be output to:
# dist/ directory
```

### Build Optimization

The production build includes:

- **Code Splitting** - Lazy loading for better performance
- **Tree Shaking** - Elimination of unused code
- **Minification** - Compressed JavaScript and CSS
- **Asset Optimization** - Optimized images and fonts
- **Source Maps** - For debugging production issues

### Static File Serving

```bash
# Option 1: Use Vite preview
npm run preview

# Option 2: Use any static file server
npx serve dist

# Option 3: Use Nginx
# Copy dist/ contents to your web server directory
```

## Deployment Options

### 1. Static Hosting (Recommended)

Deploy the `dist/` folder to any static hosting service:

- **Netlify**: Drag and drop deployment
- **Vercel**: Git-based deployment
- **GitHub Pages**: Free hosting for public repos
- **AWS S3 + CloudFront**: Scalable cloud hosting

### 2. Docker Deployment

```dockerfile
# Dockerfile
FROM nginx:alpine
COPY dist/ /usr/share/nginx/html/
EXPOSE 80
```

```bash
# Build and run
docker build -t pulseguard-dashboard .
docker run -p 80:80 pulseguard-dashboard
```

### 3. Server Deployment

```bash
# On your server
git clone your-repo
cd pulseguard-dashboard
npm install
npm run build

# Serve with Nginx or Apache
# Point document root to dist/ directory
```

## Configuration

### API Integration

The dashboard automatically configures API integration based on environment variables:

```typescript
// src/services/config.ts
const config = {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL,
  apiKey: import.meta.env.VITE_API_KEY,
  wsUrl: import.meta.env.VITE_WS_URL,
  environment: import.meta.env.VITE_ENVIRONMENT,
  debug: import.meta.env.VITE_DEBUG === 'true'
};
```

### Monitoring Configuration

Configure monitoring intervals and thresholds:

```typescript
// src/config/monitoring.ts
export const monitoringConfig = {
  refreshIntervals: {
    systemStatus: 30000,    // 30 seconds
    marketData: 60000,      // 1 minute
    apiUsage: 120000        // 2 minutes
  },
  alertThresholds: {
    apiUsageWarning: 80,    // 80% of rate limit
    missingDataThreshold: 5, // 5% missing data
    systemHealthTimeout: 10000 // 10 seconds
  }
};
```

## Troubleshooting

### Common Issues

#### 1. API Connection Failed
```bash
# Check API server status
curl http://your-api-url/health

# Verify API key
curl -H "X-API-Key: your-key" http://your-api-url/api/v1/system/status
```

#### 2. CORS Issues
Ensure your API server includes proper CORS headers:
```javascript
app.use(cors({
  origin: ['http://localhost:3001', 'https://your-dashboard-domain.com'],
  credentials: true
}));
```

#### 3. Environment Variables Not Loading
- Verify `.env` file exists in project root
- Ensure variables start with `VITE_`
- Restart development server after changes

#### 4. Build Errors
```bash
# Clear cache and rebuild
rm -rf node_modules dist
npm install
npm run build
```

### Debugging

#### Development Debugging
```bash
# Enable debug mode
VITE_DEBUG=true npm run dev

# Check browser console for detailed logs
# Check Network tab for API requests
```

#### Production Debugging
```bash
# Build with source maps
npm run build

# Check browser DevTools
# Monitor API requests in Network tab
```

## Performance Optimization

### 1. Bundle Analysis

```bash
# Install bundle analyzer
npm install --save-dev rollup-plugin-visualizer

# Analyze bundle size
npm run build -- --mode=analyze
```

### 2. Image Optimization

- Use WebP format for images
- Implement lazy loading for charts
- Optimize SVG icons

### 3. API Optimization

- Implement request caching
- Use pagination for large datasets
- Debounce frequent updates

### 4. Memory Management

- Clean up intervals on component unmount
- Use React.memo for expensive components
- Implement virtual scrolling for large lists

## Security Considerations

### 1. API Key Security
- Never commit API keys to version control
- Use environment variables for all secrets
- Rotate API keys regularly

### 2. Content Security Policy
```html
<!-- index.html -->
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; 
               connect-src 'self' http://your-api-url ws://your-api-url;
               style-src 'self' 'unsafe-inline';">
```

### 3. HTTPS Deployment
- Always use HTTPS in production
- Configure secure headers
- Implement certificate pinning

---

*Last updated: 2025-09-18*
*Version: 1.0*
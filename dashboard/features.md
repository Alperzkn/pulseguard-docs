# PulseGuard Dashboard Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Technical Stack](#technical-stack)
4. [Features](#features)
5. [Project Structure](#project-structure)
6. [Components](#components)
7. [API Integration](#api-integration)
8. [State Management](#state-management)
9. [Monitoring & Alerts](#monitoring--alerts)
10. [Development Setup](#development-setup)
11. [Environment Configuration](#environment-configuration)
12. [Deployment Guide](#deployment-guide)
13. [API Reference](#api-reference)
14. [Troubleshooting](#troubleshooting)
15. [Future Improvements](#future-improvements)

## Overview

PulseGuard Dashboard is a comprehensive real-time monitoring and visualization platform for cryptocurrency market data collection systems. It provides insights into data pipeline health, API usage, market analytics, and system performance metrics.

### Key Capabilities
- **Real-time System Monitoring**: Live tracking of data collection processes
- **Market Data Visualization**: Paginated cryptocurrency market rankings and analytics
- **API Usage Tracking**: CoinGecko API consumption monitoring with rate limit alerts
- **Data Quality Metrics**: Completeness tracking and missing data alerts
- **Alert System**: Proactive notifications for system issues and data anomalies
- **Database Health**: PostgreSQL connection status and table statistics

## Architecture

The dashboard follows a modern React SPA (Single Page Application) architecture with a clean separation of concerns:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Dashboard     │────│   PulseGuard    │────│   PostgreSQL    │
│   (React SPA)   │    │   API Server    │    │   Database      │
│                 │    │   (Node.js)     │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   User Browser  │    │   Data Collector│    │   Market Data   │
│   (Frontend)    │    │   (Cron Jobs)   │    │   (CoinGecko)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Technical Stack

### Frontend Technologies
- **React 19.1.1**: Modern functional components with hooks
- **TypeScript**: Type-safe development with strict type checking
- **Vite**: Fast build tool and development server
- **TanStack Query (React Query)**: Server state management and caching
- **Tailwind CSS**: Utility-first CSS framework
- **Lucide React**: Modern icon library
- **React Router DOM**: Client-side routing
- **Recharts**: Data visualization and charting
- **Zustand**: Lightweight state management

### Development Tools
- **ESLint**: Code linting and quality enforcement
- **TypeScript ESLint**: TypeScript-specific linting rules
- **Autoprefixer**: CSS vendor prefixing
- **PostCSS**: CSS processing

## Features

### 1. Dashboard Overview (`/`)
- **System Health Summary**: Real-time status indicators
- **Key Performance Metrics**: Data collection success rates
- **Quick Access Cards**: Navigate to detailed sections
- **Alert Notifications**: System-wide issues and warnings

### 2. Market Data (`/market`)
- **Paginated Market Rankings**: Browse all tracked cryptocurrencies
- **Real-time Price Updates**: 30-second refresh intervals
- **Market Overview Cards**: Top 4 cryptocurrencies with price changes
- **Smart Pagination**: Navigate through 1000+ coins efficiently
- **Price Formatting**: Intelligent precision based on price ranges
- **Market Cap & Volume**: Formatted display with K/M/B/T suffixes

### 3. Data Collection Monitoring (`/collection`)
- **Collection History**: Track recent data gathering operations
- **Success Rate Analytics**: Monitor collection efficiency
- **Data Completeness Tracking**: Missing data identification
- **Temporal Analysis**: Minute/hour/day data quality metrics

### 4. API Usage Monitoring (`/api`)
- **CoinGecko API Consumption**: Real-time usage tracking
- **Rate Limit Monitoring**: Prevent API throttling
- **Usage Analytics**: Historical consumption patterns
- **Cost Optimization**: Plan upgrade recommendations
- **Quota Management**: Track limits across different timeframes

### 5. System Services (`/services`)
- **Database Health**: Connection status and performance
- **Background Jobs**: Data collector process monitoring
- **System Resources**: Memory, CPU, and storage metrics
- **Service Dependencies**: External API status

### 6. Settings (`/settings`)
- **Alert Configuration**: Customize notification preferences
- **Refresh Intervals**: Adjust data polling frequencies
- **Display Preferences**: UI customization options
- **Theme Settings**: Light/dark mode toggle

## Project Structure

```
pulseguard-dashboard/
├── public/                     # Static assets
├── src/
│   ├── components/            # Reusable React components
│   │   ├── alerts/           # Alert system components
│   │   │   └── AlertToast.tsx
│   │   ├── layout/           # Layout components
│   │   │   ├── Header.tsx
│   │   │   ├── Layout.tsx
│   │   │   └── Sidebar.tsx
│   │   └── monitoring/       # Monitoring-specific components
│   │       ├── ApiUsageMonitor.tsx
│   │       ├── DataCompletenessTable.tsx
│   │       ├── MissingCoinsAlert.tsx
│   │       └── MonitoringService.tsx
│   ├── contexts/             # React Context providers
│   │   ├── AlertContext.tsx
│   │   └── SettingsContext.tsx
│   ├── pages/                # Route-level page components
│   │   ├── Api.tsx
│   │   ├── Collection.tsx
│   │   ├── Market.tsx
│   │   ├── Overview.tsx
│   │   ├── Services.tsx
│   │   └── Settings.tsx
│   ├── services/             # API client and utilities
│   │   └── apiClient.ts
│   ├── types/                # TypeScript type definitions
│   │   └── index.ts
│   ├── App.tsx               # Main application component
│   └── main.tsx              # Application entry point
├── .env                      # Environment configuration
├── package.json              # Dependencies and scripts
├── tailwind.config.js        # Tailwind CSS configuration
├── tsconfig.json             # TypeScript configuration
└── vite.config.ts            # Vite build configuration
```

## Components

### Layout Components

#### `Layout.tsx`
Main application layout wrapper providing:
- Responsive sidebar navigation
- Header with breadcrumbs
- Alert notification system integration
- Context providers wrapping

#### `Sidebar.tsx`
Navigation sidebar featuring:
- Route-based active states
- Responsive design (mobile hamburger menu)
- Icon-based navigation with lucide-react
- Clean visual hierarchy

#### `Header.tsx`
Application header including:
- Current page breadcrumbs
- User actions area
- Responsive design adaptations

### Monitoring Components

#### `DataCompletenessTable.tsx`
Data quality monitoring dashboard:
- Real-time completeness metrics
- Success rate tracking per timeframe
- Missing data identification
- Color-coded health indicators

#### `ApiUsageMonitor.tsx`
CoinGecko API consumption tracking:
- Usage percentage displays
- Rate limit warnings
- Historical usage patterns
- Plan upgrade recommendations

#### `MissingCoinsAlert.tsx`
Proactive data quality alerts:
- Missing coin detection
- Threshold-based notifications
- Dismissible alert system
- Integration with alert context

#### `MonitoringService.tsx`
Background monitoring orchestration:
- Periodic health checks
- Alert triggering logic
- Service status coordination
- Error boundary integration

### Alert System

#### `AlertToast.tsx`
Toast notification component:
- Multiple alert types (success, warning, error, info)
- Auto-dismiss functionality
- Animation transitions
- Accessible design patterns

#### `AlertContext.tsx`
Global alert state management:
- Alert queue management
- Notification lifecycle
- Context provider for app-wide access
- Type-safe alert creation

## API Integration

### `apiClient.ts`
Centralized API communication layer:

```typescript
class PulseGuardApiClient {
  private baseURL: string;
  private apiKey: string;

  // System monitoring endpoints
  async getSystemStatus(): Promise<SystemStatus>
  async getCollectionHistory(): Promise<CollectionHistory>
  async getDatabaseStats(): Promise<DatabaseStats[]>
  async testDatabaseConnection(): Promise<any>

  // Market data endpoints  
  async getTopCoins(limit: number): Promise<CoinData[]>
  async getMarketRankings(): Promise<CoinData[]>
  async getMarketRankingsPaginated(): Promise<{data: CoinData[], meta: any}>
  async getCoinHistory(): Promise<any>

  // Data quality endpoints
  async getDataCompleteness(): Promise<DataCompleteness>
  async getMissingCoins(): Promise<string[]>
  async getCoinGeckoApiUsage(): Promise<any>
}
```

### Error Handling
- Automatic retry mechanisms
- Graceful degradation for missing endpoints
- User-friendly error messages
- Fallback data strategies

### Caching Strategy
- TanStack Query for server state
- 30-second cache invalidation
- Background refetching
- Optimistic updates

## State Management

### React Query (TanStack Query)
Server state management for:
- API data caching
- Background synchronization
- Loading states
- Error handling
- Pagination state

### Zustand (Settings)
Local state management for:
- User preferences
- Theme settings
- Alert configurations
- UI state persistence

### React Context
Application-wide state:
- Alert notifications
- Settings synchronization
- Theme provider
- Authentication context

## Monitoring & Alerts

### Alert Types
1. **Data Quality Alerts**
   - Missing cryptocurrency data
   - Low data completeness rates
   - Collection failures

2. **API Usage Alerts**
   - Rate limit warnings (>80% usage)
   - Quota approaching limits
   - API key expiration

3. **System Health Alerts**
   - Database connection failures
   - Service unavailability
   - Performance degradation

### Alert Configuration
```typescript
interface AlertSettings {
  dataCompletenessThreshold: number; // Default: 95%
  apiUsageWarningThreshold: number;  // Default: 80%
  missingDataThreshold: number;      // Default: 50 coins
  enableEmailNotifications: boolean;
  enableBrowserNotifications: boolean;
}
```

## Development Setup

### Prerequisites
- Node.js 18+ (recommended 20+)
- npm or pnpm package manager
- PulseGuard API server running locally

### Local Development
```bash
# Clone repository
git clone <repository-url>
cd pulseguard-dashboard

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Edit .env with your API configuration

# Start development server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview

# Lint code
npm run lint
```

### Development Workflow
1. **Feature Development**: Create feature branches
2. **Code Quality**: ESLint + TypeScript strict mode
3. **Component Testing**: Manual testing with local API
4. **Build Verification**: Test production builds
5. **Documentation**: Update relevant docs

## Environment Configuration

### `.env` Variables
```bash
# API Configuration
VITE_API_BASE_URL=http://localhost:3000/api/v1
VITE_API_KEY=your-api-key-here

# WebSocket Configuration (optional)
VITE_WS_URL=ws://localhost:3000/ws

# Environment
VITE_ENVIRONMENT=development
```

### Environment-Specific Configs
- **Development**: Local API server, hot reloading
- **Staging**: Remote API server, debug logging
- **Production**: Optimized builds, error tracking

## Deployment Guide

### Build Process
```bash
# Install dependencies
npm install

# Build for production
npm run build

# Output in dist/ directory
```

### Deployment Options

#### 1. Static Site Hosting (Recommended)
**Vercel** (Easiest)
- Connect GitHub repository
- Automatic deployments on push
- Built-in environment variable management
- Custom domain support

**Netlify**
- Drag and drop `dist/` folder
- Git-based deployments
- Form handling and serverless functions

**AWS S3 + CloudFront**
- Upload `dist/` to S3 bucket
- Configure CloudFront distribution
- Route 53 for custom domains

#### 2. Container Deployment
**Docker**
```dockerfile
FROM node:20-alpine as builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Docker Compose**
```yaml
version: '3.8'
services:
  dashboard:
    build: .
    ports:
      - "3001:80"
    environment:
      - VITE_API_BASE_URL=https://api.pulseguard.com/api/v1
      - VITE_API_KEY=${API_KEY}
```

#### 3. VPS/Server Deployment
**Nginx Configuration**
```nginx
server {
    listen 80;
    server_name your-domain.com;
    root /var/www/pulseguard-dashboard/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### CI/CD Pipeline Example
```yaml
name: Deploy Dashboard
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '20'
      - run: npm install
      - run: npm run build
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
```

## API Reference

### Base Configuration
- **Base URL**: `/api/v1`
- **Authentication**: API Key header (`X-API-Key`)
- **Response Format**: JSON with consistent structure

### System Endpoints
```typescript
GET /system/status           // System health overview
GET /system/collections      // Collection history
GET /system/database         // Database statistics
POST /system/test-db         // Test database connection
GET /system/completeness     // Data completeness metrics
GET /system/missing-coins    // Missing cryptocurrency list
```

### Market Endpoints
```typescript
GET /market/rankings         // Paginated market rankings
GET /market/top/:count       // Top N cryptocurrencies
GET /market/summary          // Market summary statistics
GET /market/search           // Search cryptocurrencies
GET /market/range            // Get coins by rank range
GET /market/history          // Multi-coin historical data
```

### Response Format
```typescript
interface ApiResponse<T> {
  success: boolean;
  data: T | null;
  meta?: {
    timestamp: string;
    total_count?: number;
    page?: number;
    per_page?: number;
    total_pages?: number;
  };
  errors: string[] | null;
}
```

## Troubleshooting

### Common Issues

#### 1. API Connection Failed
- **Symptoms**: Dashboard shows "No data available"
- **Solutions**: 
  - Check API server is running
  - Verify environment variables
  - Confirm API key validity
  - Check network connectivity

#### 2. Build Failures
- **Symptoms**: TypeScript or build errors
- **Solutions**:
  - Update dependencies: `npm update`
  - Clear cache: `rm -rf node_modules/.cache`
  - Reinstall: `rm -rf node_modules && npm install`

#### 3. Missing Market Data
- **Symptoms**: Empty market rankings table
- **Solutions**:
  - Check database connection
  - Verify data collection processes
  - Review API usage limits
  - Check database table contents

#### 4. Performance Issues
- **Symptoms**: Slow loading, high memory usage
- **Solutions**:
  - Reduce polling intervals
  - Implement pagination where missing
  - Optimize query caching
  - Monitor network requests

### Debug Mode
Enable debug logging:
```javascript
localStorage.setItem('debug', 'pulseguard:*')
```

## Future Improvements

### Planned Features
1. **Enhanced Analytics**
   - Price prediction models
   - Historical trend analysis
   - Portfolio tracking capabilities

2. **Advanced Alerts**
   - Email notification integration
   - Slack/Discord webhooks
   - Custom alert rules engine
   - Mobile push notifications

3. **Performance Optimization**
   - Virtual scrolling for large datasets
   - GraphQL integration
   - Service worker caching
   - Progressive Web App (PWA) features

4. **User Management**
   - Multi-user support
   - Role-based permissions
   - Audit logging
   - SSO integration

5. **Data Export**
   - CSV/Excel export functionality
   - PDF report generation
   - Scheduled reports
   - API data export

### Technical Debt
- Migrate to React Server Components
- Implement comprehensive testing suite
- Add error boundary components
- Improve TypeScript coverage
- Add internationalization (i18n)

## Support & Maintenance

### Monitoring Checklist
- [ ] API endpoint health
- [ ] Database connection status
- [ ] Data collection processes
- [ ] Alert system functionality
- [ ] UI responsiveness
- [ ] Error rates and logs

### Update Process
1. Review dependency updates monthly
2. Test in staging environment
3. Deploy during maintenance windows
4. Monitor post-deployment metrics
5. Rollback plan for critical issues

### Documentation Updates
- Keep API reference current
- Update deployment guides
- Document new features
- Maintain troubleshooting guides

---

*Last Updated: 2025-09-11*
*Version: 1.0.0*
*Maintainer: PulseGuard Development Team*
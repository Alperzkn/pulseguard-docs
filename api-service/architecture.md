# PulseGuard API Architecture

## Overview

The PulseGuard API is a RESTful service that provides access to cryptocurrency price data collected by the PulseGuard data collector. It serves as the backend API for applications that need real-time and historical cryptocurrency market data.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PulseGuard Ecosystem                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────┐ │
│  │   Data          │    │   API Service   │    │Dashboard │ │
│  │   Collector     │    │   (This App)    │    │   UI     │ │
│  │                 │    │                 │    │          │ │
│  │ - CoinGecko API │───▶│ - REST API      │───▶│ - React  │ │
│  │ - Data Storage  │    │ - Authentication│    │ - Charts │ │
│  │ - Aggregation   │    │ - Rate Limiting │    │ - Admin  │ │
│  └─────────────────┘    └─────────────────┘    └──────────┘ │
│           │                       │                         │
│           ▼                       ▼                         │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              PostgreSQL Database                        │ │
│  │ - minute_close   - coin_priorities                      │ │
│  │ - hourly_close   - collection_status                    │ │
│  │ - daily_close    - api_usage_stats                      │ │
│  │ - weekly_close   - system_metrics                       │ │
│  │ - monthly_close                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## API Service Architecture

### 1. Application Layer (src/)

```
src/
├── server.js              # Express app configuration
├── config/
│   └── database.js         # Database connection management
├── controllers/            # HTTP request handlers
│   ├── CoinsController.js  # Individual coin operations
│   ├── MarketController.js # Market data and rankings
│   ├── SystemController.js # Health and system status
│   └── ApiController.js    # API usage monitoring
├── models/                 # Database interaction layer
│   ├── CoinsModel.js       # Coin data queries
│   ├── MarketModel.js      # Market data queries
│   ├── SystemModel.js      # System status queries
│   └── ApiModel.js         # API usage tracking
├── middleware/             # Express middleware
│   ├── auth.js            # API key authentication
│   ├── rateLimiter.js     # Request rate limiting
│   └── validation.js      # Input validation
└── routes/                # API route definitions
    ├── coins.js           # /api/v1/coins/*
    ├── market.js          # /api/v1/market/*
    ├── system.js          # /api/v1/system/*
    └── api.js             # /api/v1/api/*
```

### 2. Data Access Layer

#### Database Tables

| Table | Purpose | Retention | Indexes |
|-------|---------|-----------|---------|
| `minute_close` | Real-time prices | 60 minutes | coin_id, minute_ts |
| `hourly_close` | Hourly aggregates | 24 hours | coin_id, hour_ts |
| `daily_close` | Daily aggregates | 7 days | coin_id, day_ts |
| `weekly_close` | Weekly aggregates | 52 weeks | coin_id, week_ts |
| `monthly_close` | Monthly aggregates | Permanent | coin_id, month_ts |
| `coin_priorities` | Market rankings | Current | market_cap_rank |
| `collection_status` | System health | 30 days | start_time |

#### Query Optimization

- **Time-based partitioning** for efficient historical queries
- **Composite indexes** on (coin_id, timestamp) for fast filtering
- **Market cap ranking indexes** for top-N queries
- **Connection pooling** for concurrent request handling

### 3. Security Layer

#### Authentication
- **API Key Based**: Simple header-based authentication
- **Environment Variables**: Secure key storage
- **No JWT Overhead**: Optimized for high-frequency API usage

#### Rate Limiting
- **Tiered Limits**: Different limits for different endpoint types
- **IP-based Tracking**: Per-client request counting
- **Graceful Degradation**: Clear error messages and retry headers

#### Input Validation
- **Schema Validation**: Request parameter validation
- **SQL Injection Prevention**: Parameterized queries
- **XSS Protection**: Input sanitization

### 4. Performance Layer

#### Caching Strategy
- **Database Query Caching**: Frequently accessed data
- **Response Caching**: Static market data
- **Connection Pooling**: Reused database connections

#### Response Optimization
- **Pagination**: Large dataset handling
- **Selective Fields**: Minimize payload size
- **Compression**: Gzip response compression

## API Design Principles

### 1. RESTful Architecture
- **Resource-based URLs**: `/coins/{id}/history`
- **HTTP Methods**: GET for data retrieval
- **Status Codes**: Proper HTTP response codes
- **Consistent Naming**: snake_case for parameters

### 2. Response Structure
```json
{
  "success": true,
  "data": { /* actual data */ },
  "meta": {
    "timestamp": "2025-09-18T10:30:00Z",
    "total_count": 1000,
    "page": 1,
    "limit": 100
  },
  "errors": null
}
```

### 3. Error Handling
- **Consistent Error Format**: Structured error responses
- **Meaningful Messages**: Clear error descriptions
- **Error Codes**: Application-specific error codes
- **Logging**: Comprehensive error logging

## Deployment Architecture

### 1. Container Strategy
```dockerfile
# Multi-stage build for optimization
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
EXPOSE 3000
CMD ["npm", "start"]
```

### 2. Service Communication
- **Database**: Direct PostgreSQL connection
- **Health Checks**: Built-in health monitoring
- **Logging**: Structured JSON logs
- **Metrics**: Performance and usage metrics

### 3. Monitoring
- **Health Endpoints**: `/health` for basic health checks
- **System Status**: Real-time collection monitoring
- **API Metrics**: Request rates and response times
- **Database Metrics**: Connection pool and query performance

## Integration Patterns

### 1. For Frontend Applications
```javascript
// React/Vue.js integration
const apiClient = new PulseGuardAPI({
  baseURL: process.env.REACT_APP_API_URL,
  apiKey: process.env.REACT_APP_API_KEY
});
```

### 2. For Backend Services
```javascript
// Express.js integration
const market = await pulseguardAPI.market.getTop(10);
const prices = await pulseguardAPI.coins.getHistory('bitcoin', {
  timeframe: 'hour',
  limit: 24
});
```

### 3. For Data Analysis
```python
# Python integration
import requests

api = PulseGuardAPI(
    base_url=os.getenv('PULSEGUARD_API_URL'),
    api_key=os.getenv('PULSEGUARD_API_KEY')
)
```

## Scaling Considerations

### 1. Horizontal Scaling
- **Stateless Design**: No server-side sessions
- **Load Balancing**: Multiple API instances
- **Database Scaling**: Read replicas for queries

### 2. Performance Optimization
- **Query Optimization**: Efficient database queries
- **Caching Layers**: Redis for frequently accessed data
- **CDN Integration**: Static asset delivery

### 3. Monitoring and Alerting
- **Application Metrics**: Response times and error rates
- **Infrastructure Metrics**: CPU, memory, and disk usage
- **Business Metrics**: API usage and data freshness

## Data Flow

### 1. Collection Process
```
CoinGecko API → PulseGuard Collector → PostgreSQL → API Service → Client
```

### 2. API Request Flow
```
Client Request → Auth Middleware → Rate Limiter → Controller → Model → Database → Response
```

### 3. Error Flow
```
Error → Logger → Error Handler → Formatted Response → Client
```

---

*Last updated: 2025-09-18*
*Version: 1.0*
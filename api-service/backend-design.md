# PulseGuard Backend API Service Design

## Overview

This document outlines the architecture and implementation plan for a backend API service that will serve cryptocurrency data collected by the PulseGuard system. The API will provide REST endpoints to access multi-timeframe price data, market rankings, and system status.

## Current Data Structure Analysis

### Database Tables
- **minute_close** - Real-time minute data (60 min retention)
- **hourly_close** - Hourly aggregated data (24 hour retention)  
- **daily_close** - Daily aggregated data (7 day retention)
- **weekly_close** - Weekly aggregated data (52 week retention)
- **monthly_close** - Monthly aggregated data (permanent)
- **coin_priorities** - Market cap rankings and metadata
- **collection_status** - System health and collection metrics

### Data Fields Available
- Price data: `price_close`, `asset_id`, timestamps
- Market data: `market_cap_rank`, `market_cap_usd`, `trading_volume_24h`
- System data: Collection status, error tracking, batch details

## Architecture Design

### 1. Technology Stack
```
Backend Framework: Express.js (Node.js)
Database: PostgreSQL (existing PulseGuard database)
Cache Layer: Redis (optional for performance)
Authentication: JWT tokens (optional)
Documentation: Swagger/OpenAPI
Deployment: Docker container alongside PulseGuard
```

### 2. Service Structure
```
pulseguard-api/
├── src/
│   ├── controllers/        # API endpoint handlers
│   │   ├── CoinsController.js
│   │   ├── MarketController.js
│   │   ├── SystemController.js
│   │   └── ApiController.js      # NEW: API usage monitoring
│   ├── services/           # Business logic layer
│   ├── models/             # Database models/queries
│   │   └── SystemModel.js        # System status and metrics
│   ├── middleware/         # Auth, rate limiting, validation
│   │   ├── auth.js              # API key authentication
│   │   └── validation.js        # Input validation
│   ├── routes/             # Route definitions
│   │   ├── coins.js
│   │   ├── market.js
│   │   ├── system.js
│   │   └── api.js               # NEW: API usage routes
│   ├── utils/              # Helper functions
│   └── config/             # Configuration management
├── tests/                  # Unit and integration tests
├── docs/                   # API documentation
├── Dockerfile
└── package.json
```

### 3. API Endpoints Design

#### 3.1 Market Data Endpoints

**GET /api/v1/coins**
- List all available cryptocurrencies with basic info
- Query params: `limit`, `offset`, `sort_by` (rank/name)
- Response: Array of coins with id, name, rank, last_price

**GET /api/v1/coins/{coin_id}**
- Get detailed information for specific cryptocurrency
- Response: Coin metadata, current price, market cap, volume

**GET /api/v1/coins/{coin_id}/price**
- Get current/latest price for specific coin
- Query params: `timeframe` (minute/hour/day/week/month)
- Response: Latest price data with timestamp

#### 3.2 Historical Data Endpoints

**GET /api/v1/coins/{coin_id}/history**
- Get historical price data for specific coin
- Query params: 
  - `timeframe` (minute/hour/day/week/month)
  - `from` (start timestamp)
  - `to` (end timestamp) 
  - `limit` (max records)
- Response: Array of price points with timestamps

**GET /api/v1/market/history**
- Get historical data for multiple coins
- Query params: `coin_ids[]`, `timeframe`, `from`, `to`, `limit`
- Response: Object with coin_id keys and price arrays

#### 3.3 Market Rankings Endpoints

**GET /api/v1/market/rankings**
- Get current market cap rankings
- Query params: `limit`, `offset`
- Response: Ranked list with market cap, volume, price changes

**GET /api/v1/market/top/{count}**
- Get top N cryptocurrencies by market cap
- Response: Array of top coins with full market data

#### 3.4 System Status Endpoints

**GET /api/v1/system/status**
- Get overall system health and collection status
- Response: Last collection time, success rates, system metrics

**GET /api/v1/system/collections**
- Get collection history and statistics
- Query params: `from`, `to`, `limit`
- Response: Array of collection runs with success/failure details

**GET /api/v1/system/completeness**
- Get data completeness metrics across timeframes
- Response: Expected vs actual coin counts, success rates per timeframe

**GET /api/v1/system/missing-coins**
- Get list of coins missing from latest collection
- Response: Array of coin IDs that failed to collect in latest run

#### 3.5 API Management Endpoints

**GET /api/v1/api/coingecko-usage**
- Get CoinGecko API usage statistics and recommendations
- Response: Real-time usage calculations based on actual collection system
- Features:
  - Current usage across all timeframes (minute/hour/day/month)
  - Usage percentage against CoinGecko API limits
  - Calculation details showing batching strategy (250 coins per call)
  - Upgrade recommendations with current pricing tiers
  - Collection strategy breakdown (every minute via cron)

### 4. Response Format Standards

```json
{
  "success": true,
  "data": {...},
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z",
    "total_count": 1000,
    "page": 1,
    "limit": 50
  },
  "errors": null
}
```

### 5. Performance Considerations

#### 5.1 Database Optimization
- Use existing indexes on coin_rank and timestamp columns
- Implement database connection pooling
- Add query result caching for frequently requested data
- Use read replicas if needed for scaling

#### 5.2 Caching Strategy
- Cache coin metadata (updates daily)
- Cache current prices (updates every minute)
- Cache historical data (immutable after retention periods)
- Implement cache invalidation for real-time data

#### 5.3 Rate Limiting
- Implement rate limiting per IP/API key
- Different rate limits for different endpoint types
- Burst allowances for authenticated users

### 6. Security Considerations

#### 6.1 Authentication - API Key Based

**Implementation Approach:**
- Use `X-API-Key` header for all requests
- Simple, stateless authentication suitable for service-to-service communication
- Rate limiting tied to API key

**API Key Middleware:**
```javascript
const validateApiKey = (req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  
  if (!apiKey) {
    return res.status(401).json({
      success: false,
      errors: ['API key required']
    });
  }
  
  if (apiKey !== process.env.API_KEY_SECRET) {
    return res.status(401).json({
      success: false,
      errors: ['Invalid API key']
    });
  }
  
  next();
};
```

**For detailed integration instructions, see: `API_INTEGRATION_GUIDE.md`**

#### 6.2 Data Validation
- Input validation for all parameters
- SQL injection prevention (parameterized queries)
- CORS configuration for web applications

### 7. Monitoring and Logging

#### 7.1 Application Metrics
- Request/response times
- Error rates by endpoint
- Database query performance
- Cache hit/miss rates

#### 7.2 Business Metrics
- Most requested coins
- Popular timeframes
- Data freshness tracking
- API usage patterns

### 8. Deployment Architecture - Separate Container Approach

#### 8.1 DigitalOcean Droplet Deployment

**Local Development Setup:**
```
/Users/alperzkn/Documents/dev/
├── pulseguard/                 # Existing data collector project
│   ├── app/
│   ├── deploy-droplet.sh
│   └── Dockerfile
└── pulseguard-api/             # NEW FOLDER - Create separate API project
    ├── src/
    │   ├── server.js           # Main Express server
    │   ├── routes/             # API route definitions
    │   ├── controllers/        # Business logic
    │   ├── models/             # Database queries
    │   ├── middleware/         # Auth, validation, rate limiting
    │   └── config/             # Configuration management
    ├── deploy-api.sh           # Deployment script
    ├── Dockerfile              # API container definition
    ├── package.json            # API dependencies
    └── .env.api               # API configuration
```

**Droplet Structure After Deployment:**
```
/home/user/
├── pulseguard/                 # Existing data collector
└── pulseguard-api/             # New API service (separate container)
```

#### 8.2 API Service Dockerfile
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY src/ ./src/
COPY docs/ ./docs/

EXPOSE 3000

CMD ["node", "src/server.js"]
```

#### 8.3 API Deployment Script (deploy-api.sh)
```bash
#!/bin/bash

set -e

IMAGE_NAME="crypto-api"
CONTAINER_NAME="crypto-api-service"
ENV_FILE=".env.api"
PORT="3000"

case "$1" in
  deploy)
    echo "🚀 Deploying API service to DigitalOcean droplet..."
    
    # Stop and remove existing container
    docker stop $CONTAINER_NAME 2>/dev/null || true
    docker rm $CONTAINER_NAME 2>/dev/null || true
    
    # Build new image
    echo "📦 Building API service image..."
    docker build -t $IMAGE_NAME .
    
    # Run new container
    echo "🔄 Starting API service container..."
    docker run -d \
        --name $CONTAINER_NAME \
        --restart=unless-stopped \
        --network=host \
        --env-file $ENV_FILE \
        -v $(pwd)/logs:/app/logs \
        $IMAGE_NAME
    
    echo "✅ API service deployed successfully!"
    echo "🌐 API available at: http://your-droplet-ip:3000"
    ;;
  stop)
    docker stop $CONTAINER_NAME
    ;;
  logs)
    docker logs -f $CONTAINER_NAME
    ;;
  *)
    echo "Usage: $0 {deploy|stop|logs}"
    exit 1
    ;;
esac
```

#### 8.4 Environment Configuration (.env.api)
```bash
# API Service Configuration
NODE_ENV=production
PORT=3000
API_VERSION=v1

# Database (same as data collector)
DATABASE_URL=postgresql://username:password@localhost:5432/crypto_db

# Authentication
JWT_SECRET=your-super-secret-jwt-key
API_KEY_SECRET=your-api-key-secret

# Rate Limiting
RATE_LIMIT_WINDOW_MS=900000  # 15 minutes
RATE_LIMIT_MAX_REQUESTS=100  # requests per window

# Caching
REDIS_URL=redis://localhost:6379
CACHE_TTL_SECONDS=300        # 5 minutes

# Logging
LOG_LEVEL=info
LOG_FILE=/app/logs/api.log
```

#### 8.5 Implementation Steps

**Step 1: Create Local API Project**
```bash
# Create separate folder for API service
mkdir /Users/alperzkn/Documents/dev/pulseguard-api
cd /Users/alperzkn/Documents/dev/pulseguard-api

# Initialize new Node.js project
npm init -y
```

**Step 2: Development Phase**
- Build Express.js API service following the architecture in this document
- Test locally against the same database as PulseGuard collector
- Implement all endpoints and middleware

**Step 3: Deployment to Droplet**
```bash
# On droplet - clone API service repository (separate from data collector)
git clone your-api-repo.git pulseguard-api
cd pulseguard-api

# Configure environment
cp .env.api.example .env.api
# Edit .env.api with your database credentials

# Deploy API service
chmod +x deploy-api.sh
./deploy-api.sh deploy

# Verify both services are running
docker ps
# Should show both: crypto-collector-continuous AND crypto-api-service

# Test API
curl http://your-droplet-ip:3000/api/v1/system/status
```

#### 8.6 Service Communication
```
Your App/Frontend <--HTTP--> API Service Container (Port 3000)
                                      |
                                      v
                                Database (PostgreSQL)
                                      ^
                                      |
                           Data Collector Container
```

#### 8.7 Load Balancing & Scaling (Optional)
- Run multiple API containers on different ports (3000, 3001, 3002)
- Use nginx reverse proxy for load balancing
- Scale API independently from data collection

### 9. Development Phases

#### Phase 1: Core API (Week 1)
- Basic Express.js setup with database connection
- Core endpoints: /coins, /coins/{id}, /coins/{id}/price
- Simple response formatting
- Basic error handling

#### Phase 2: Historical Data (Week 2)
- Historical data endpoints with timeframe support
- Query optimization and caching
- Comprehensive testing
- API documentation

#### Phase 3: Advanced Features (Week 3)
- Market rankings and top coins endpoints
- System status and monitoring endpoints
- Rate limiting and security features
- Performance optimization

#### Phase 4: Production Ready (Week 4)
- Authentication system
- Comprehensive monitoring
- Load testing and optimization
- Production deployment setup

### 10. Example Integration

#### Frontend Application Usage
```javascript
// Get top 10 cryptocurrencies
const topCoins = await fetch('/api/v1/market/top/10');

// Get Bitcoin price history (last 24 hours)
const btcHistory = await fetch('/api/v1/coins/bitcoin/history?timeframe=hour&limit=24');

// Get current prices for multiple coins
const prices = await fetch('/api/v1/market/history?coin_ids[]=bitcoin&coin_ids[]=ethereum&timeframe=minute&limit=1');
```

## Benefits of This Architecture

1. **Separation of Concerns**: Data collection (PulseGuard) and data serving (API) are separate services
2. **Scalability**: API service can be scaled independently based on demand
3. **Performance**: Optimized for read operations with caching and indexing
4. **Flexibility**: Multiple timeframes and query options for different use cases
5. **Maintainability**: Clean architecture with proper error handling and monitoring
6. **Integration Ready**: RESTful API that can serve web apps, mobile apps, and other services

## Next Steps

1. Review this design and provide feedback
2. Set up basic Express.js project structure
3. Implement core database models and queries
4. Build and test core API endpoints
5. Add caching and performance optimizations
6. Deploy alongside existing PulseGuard system

Would you like me to proceed with implementing this API service based on your feedback?
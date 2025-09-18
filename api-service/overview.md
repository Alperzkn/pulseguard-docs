# PulseGuard API Service

A high-performance REST API for serving cryptocurrency price data collected by the PulseGuard data collector system.

## Features

- 🔐 **API Key Authentication** - Secure access with simple header-based auth
- 📊 **Multi-timeframe Data** - Minute, hourly, daily, weekly, monthly price data
- 🎯 **Market Rankings** - Real-time market cap rankings and top coins
- 🚀 **High Performance** - Optimized database queries and response caching
- 📈 **Historical Data** - Comprehensive price history with flexible filtering
- 🛡️ **Rate Limiting** - Built-in request throttling and abuse prevention
- 🏥 **Health Monitoring** - System status and collection monitoring endpoints
- 🐳 **Docker Ready** - Containerized deployment with health checks

## Quick Start

### 1. Setup Environment

```bash
# Copy environment template
cp .env.api.example .env.api

# Edit configuration
nano .env.api
```

Required configuration:
```bash
# Database connection (same as PulseGuard collector)
DATABASE_URL=postgresql://username:password@localhost:5432/crypto_db

# API authentication  
API_KEY_SECRET=your-super-secret-api-key-here

# Optional: Rate limiting
RATE_LIMIT_MAX_REQUESTS=100
```

### 2. Install Dependencies

```bash
npm install
```

### 3. Run Locally (Development)

```bash
# Start with auto-reload
npm run dev

# Or start normally
npm start
```

API will be available at `http://localhost:3000/api/v1`

### 4. Deploy with Docker

```bash
# Deploy to production
./deploy-api.sh deploy

# View logs
./deploy-api.sh logs

# Check status
./deploy-api.sh status

# Test endpoints
./deploy-api.sh test
```

## API Endpoints

### Authentication

All endpoints (except `/health`) require API key authentication:

```bash
curl -H "X-API-Key: your-api-key" http://localhost:3000/api/v1/coins
```

### Core Endpoints

#### Coins
- `GET /api/v1/coins` - List all cryptocurrencies
- `GET /api/v1/coins/{id}` - Get specific coin details
- `GET /api/v1/coins/{id}/price` - Get current price
- `GET /api/v1/coins/{id}/history` - Get price history

#### Market
- `GET /api/v1/market/rankings` - Market cap rankings
- `GET /api/v1/market/top/{count}` - Top N coins
- `GET /api/v1/market/summary` - Market statistics
- `GET /api/v1/market/history` - Multi-coin history

#### System
- `GET /api/v1/system/status` - Collection system status
- `GET /api/v1/system/health` - Health check (no auth)
- `GET /api/v1/system/collections` - Collection history

### Example Requests

```bash
# Get top 10 cryptocurrencies
curl -H "X-API-Key: your-key" \
  "http://localhost:3000/api/v1/market/top/10"

# Get Bitcoin price history (last 24 hours)
curl -H "X-API-Key: your-key" \
  "http://localhost:3000/api/v1/coins/bitcoin/history?timeframe=hour&limit=24"

# Get current prices for multiple coins
curl -H "X-API-Key: your-key" \
  "http://localhost:3000/api/v1/market/history?coin_ids=bitcoin,ethereum&timeframe=minute&limit=1"
```

## Response Format

All API responses follow this structure:

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z",
    "total_count": 1000,
    "page": 1
  },
  "errors": null
}
```

## Rate Limiting

- **Standard**: 100 requests per 15 minutes
- **Basic operations**: 200 requests per 15 minutes  
- **Health checks**: 100 requests per minute

Rate limit headers included in responses:
- `RateLimit-Limit` - Request limit
- `RateLimit-Remaining` - Remaining requests
- `RateLimit-Reset` - Reset time

## Error Handling

HTTP Status Codes:
- `200` - Success
- `400` - Bad Request (validation error)
- `401` - Unauthorized (invalid API key)
- `404` - Not Found
- `429` - Too Many Requests (rate limited)
- `500` - Internal Server Error

## Deployment

### DigitalOcean Droplet

1. **Clone repository on droplet:**
```bash
git clone your-repo-url pulseguard-api
cd pulseguard-api
```

2. **Configure environment:**
```bash
cp .env.api.example .env.api
# Edit .env.api with your database credentials
```

3. **Deploy:**
```bash
./deploy-api.sh deploy
```

4. **Verify deployment:**
```bash
# Check both services are running
docker ps

# Should show:
# pulseguard-api-service (port 3000)
# crypto-collector-continuous (your existing collector)
```

### Service Management

```bash
# View API service logs
./deploy-api.sh logs

# Restart API service
./deploy-api.sh restart

# Check health
./deploy-api.sh status

# Test endpoints
./deploy-api.sh test
```

## Project Structure

```
src/
├── server.js              # Main Express application
├── config/
│   └── database.js         # Database connection
├── controllers/            # Request handlers
│   ├── CoinsController.js
│   ├── MarketController.js
│   └── SystemController.js
├── models/                 # Database queries
│   ├── CoinsModel.js
│   ├── MarketModel.js
│   └── SystemModel.js
├── middleware/             # Express middleware
│   ├── auth.js            # API key authentication
│   ├── rateLimiter.js     # Rate limiting
│   └── validation.js      # Input validation
└── routes/                # Route definitions
    ├── coins.js
    ├── market.js
    └── system.js
```

## Development

### Local Development

```bash
# Install dependencies
npm install

# Start with auto-reload
npm run dev

# Run tests
npm test
```

### Environment Variables

See `.env.api.example` for all configuration options.

Key settings:
- `DATABASE_URL` - PostgreSQL connection string
- `API_KEY_SECRET` - Authentication key
- `RATE_LIMIT_MAX_REQUESTS` - Rate limiting
- `NODE_ENV` - Environment (development/production)

## Integration

For detailed integration instructions, see `docs/API_INTEGRATION_GUIDE.md`.

Quick integration example:

```javascript
const apiClient = axios.create({
  baseURL: 'http://your-droplet-ip:3000/api/v1',
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

// Get top cryptocurrencies
const topCoins = await apiClient.get('/market/top/10');
```

## Monitoring

The API includes comprehensive monitoring:

- **Health checks**: `/health` endpoint
- **System status**: Real-time collection status
- **Database stats**: Table sizes and performance
- **Rate limiting**: Request tracking and throttling

## Security

- API key authentication for all endpoints
- Rate limiting to prevent abuse
- Input validation and sanitization
- CORS configuration
- Security headers via Helmet.js
- No sensitive data in error responses

## Support

1. Check logs: `./deploy-api.sh logs`
2. Verify database connectivity
3. Test with health endpoint: `curl http://localhost:3000/health`
4. Check rate limits and API key validity

For detailed troubleshooting, see the integration guide.
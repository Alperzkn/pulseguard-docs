# API Documentation

## Overview

The PulseGuard API provides RESTful endpoints for accessing cryptocurrency price data, market information, and system status. All endpoints require API key authentication.

## Base URL

```
Production: https://your-server.com/api/v1
Development: http://localhost:3000/api/v1
```

## Authentication

All API endpoints (except `/health`) require authentication using an API key in the request header:

```bash
curl -H "X-API-Key: your-api-key" https://your-server.com/api/v1/endpoint
```

## Response Format

All API responses follow this consistent structure:

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z",
    "total_count": 1000,
    "page": 1,
    "limit": 100
  },
  "errors": null
}
```

## Rate Limiting

- **Standard Endpoints**: 100 requests per 15 minutes
- **Health Endpoint**: 100 requests per minute
- **Heavy Operations**: 50 requests per 15 minutes

Rate limit headers are included in all responses:
- `RateLimit-Limit`: Request limit for the current window
- `RateLimit-Remaining`: Remaining requests in current window  
- `RateLimit-Reset`: Time when rate limit resets

## Endpoints

### System Endpoints

#### Health Check
```http
GET /health
```

**Description**: Check API service health (no authentication required)

**Response**:
```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "timestamp": "2025-01-15T10:30:00Z",
    "uptime": 3600,
    "version": "1.0.0"
  }
}
```

#### System Status
```http
GET /system/status
```

**Description**: Get comprehensive system health and metrics

**Response**:
```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "uptime": 3600,
    "last_collection": "2025-01-15T10:25:00Z",
    "collections_today": 1440,
    "success_rate": 99.5,
    "database_status": "connected",
    "api_usage": {
      "calls_today": 28750,
      "rate_limit_remaining": 1250
    }
  }
}
```

#### Collection History
```http
GET /system/collections
```

**Query Parameters**:
- `limit` (integer): Number of records to return (default: 50, max: 500)
- `offset` (integer): Number of records to skip (default: 0)
- `status` (string): Filter by status (`running`, `completed`, `failed`)
- `start_date` (string): Start date filter (ISO format)
- `end_date` (string): End date filter (ISO format)

**Example**:
```bash
GET /system/collections?limit=10&status=completed
```

**Response**:
```json
{
  "success": true,
  "data": [
    {
      "id": 12345,
      "collection_type": "complete",
      "start_time": "2025-01-15T10:20:00Z",
      "end_time": "2025-01-15T10:25:00Z",
      "coins_processed": 1000,
      "coins_failed": 5,
      "status": "completed",
      "success_rate": 99.5
    }
  ],
  "meta": {
    "total_count": 2876,
    "page": 1,
    "limit": 10
  }
}
```

#### Database Health
```http
GET /system/database
```

**Description**: Get database connection status and table statistics

**Response**:
```json
{
  "success": true,
  "data": {
    "status": "connected",
    "connection_pool": {
      "active_connections": 5,
      "idle_connections": 15,
      "total_connections": 20
    },
    "tables": {
      "minute_close": {
        "row_count": 2500000,
        "size_mb": 1250.5
      },
      "hourly_close": {
        "row_count": 120000,
        "size_mb": 45.2
      },
      "daily_close": {
        "row_count": 7000,
        "size_mb": 2.8
      }
    }
  }
}
```

### Coin Endpoints

#### List All Coins
```http
GET /coins
```

**Query Parameters**:
- `limit` (integer): Number of coins to return (default: 100, max: 1000)
- `offset` (integer): Number of coins to skip (default: 0)
- `search` (string): Search by coin name or symbol
- `rank_min` (integer): Minimum market cap rank
- `rank_max` (integer): Maximum market cap rank

**Example**:
```bash
GET /coins?limit=50&search=bitcoin
```

**Response**:
```json
{
  "success": true,
  "data": [
    {
      "coin_id": "bitcoin",
      "symbol": "BTC",
      "name": "Bitcoin",
      "current_price": 45250.75,
      "market_cap_rank": 1,
      "market_cap": 890500000000,
      "last_updated": "2025-01-15T10:30:00Z"
    }
  ]
}
```

#### Get Coin Details
```http
GET /coins/{coin_id}
```

**Parameters**:
- `coin_id` (string): Unique coin identifier (e.g., "bitcoin")

**Response**:
```json
{
  "success": true,
  "data": {
    "coin_id": "bitcoin",
    "symbol": "BTC",
    "name": "Bitcoin",
    "current_price": 45250.75,
    "market_cap": 890500000000,
    "market_cap_rank": 1,
    "volume_24h": 28500000000,
    "price_change_24h": 2.5,
    "price_change_percentage_24h": 5.85,
    "last_updated": "2025-01-15T10:30:00Z"
  }
}
```

#### Get Coin Price History
```http
GET /coins/{coin_id}/history
```

**Parameters**:
- `coin_id` (string): Unique coin identifier

**Query Parameters**:
- `timeframe` (string): Data granularity (`minute`, `hour`, `day`, `week`, `month`)
- `limit` (integer): Number of data points (default: 100, max: 1000)
- `start_date` (string): Start date (ISO format)
- `end_date` (string): End date (ISO format)

**Example**:
```bash
GET /coins/bitcoin/history?timeframe=hour&limit=24
```

**Response**:
```json
{
  "success": true,
  "data": [
    {
      "timestamp": "2025-01-15T10:00:00Z",
      "price_usd": 45250.75,
      "volume_24h": 28500000000,
      "market_cap": 890500000000
    }
  ],
  "meta": {
    "timeframe": "hour",
    "total_count": 24,
    "start_date": "2025-01-14T10:00:00Z",
    "end_date": "2025-01-15T10:00:00Z"
  }
}
```

### Market Endpoints

#### Market Summary
```http
GET /market/summary
```

**Description**: Get overall market statistics

**Response**:
```json
{
  "success": true,
  "data": {
    "total_market_cap": 2500000000000,
    "total_volume_24h": 95000000000,
    "bitcoin_dominance": 42.5,
    "active_cryptocurrencies": 1000,
    "market_cap_change_24h": 2.8,
    "updated_at": "2025-01-15T10:30:00Z"
  }
}
```

#### Top Cryptocurrencies
```http
GET /market/top/{count}
```

**Parameters**:
- `count` (integer): Number of top coins to return (max: 1000)

**Query Parameters**:
- `convert` (string): Currency to convert prices to (default: "usd")

**Example**:
```bash
GET /market/top/10
```

**Response**:
```json
{
  "success": true,
  "data": [
    {
      "rank": 1,
      "coin_id": "bitcoin",
      "symbol": "BTC",
      "name": "Bitcoin",
      "price_usd": 45250.75,
      "market_cap": 890500000000,
      "volume_24h": 28500000000,
      "price_change_24h": 5.85
    }
  ]
}
```

#### Market Rankings
```http
GET /market/rankings
```

**Query Parameters**:
- `page` (integer): Page number (default: 1)
- `limit` (integer): Items per page (default: 100, max: 250)
- `sort` (string): Sort by (`market_cap`, `volume`, `price_change_24h`)
- `order` (string): Sort order (`asc`, `desc`)

**Response**:
```json
{
  "success": true,
  "data": [
    {
      "rank": 1,
      "coin_id": "bitcoin",
      "symbol": "BTC",
      "name": "Bitcoin",
      "price_usd": 45250.75,
      "market_cap": 890500000000,
      "volume_24h": 28500000000
    }
  ],
  "meta": {
    "page": 1,
    "limit": 100,
    "total_count": 1000,
    "total_pages": 10
  }
}
```

#### Multi-Coin History
```http
GET /market/history
```

**Query Parameters**:
- `coin_ids` (string): Comma-separated list of coin IDs
- `timeframe` (string): Data granularity (`minute`, `hour`, `day`, `week`, `month`)
- `limit` (integer): Number of data points per coin
- `start_date` (string): Start date (ISO format)
- `end_date` (string): End date (ISO format)

**Example**:
```bash
GET /market/history?coin_ids=bitcoin,ethereum&timeframe=day&limit=7
```

**Response**:
```json
{
  "success": true,
  "data": {
    "bitcoin": [
      {
        "timestamp": "2025-01-15T00:00:00Z",
        "price_usd": 45250.75,
        "volume_24h": 28500000000
      }
    ],
    "ethereum": [
      {
        "timestamp": "2025-01-15T00:00:00Z",
        "price_usd": 3250.50,
        "volume_24h": 18500000000
      }
    ]
  }
}
```

## Error Responses

### Error Format

All errors follow this format:

```json
{
  "success": false,
  "data": null,
  "errors": [
    {
      "code": "INVALID_API_KEY",
      "message": "Invalid or missing API key",
      "field": "X-API-Key"
    }
  ],
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

### HTTP Status Codes

| Code | Description | Common Causes |
|------|-------------|---------------|
| 200 | OK | Request successful |
| 400 | Bad Request | Invalid parameters, malformed request |
| 401 | Unauthorized | Missing or invalid API key |
| 404 | Not Found | Endpoint or resource not found |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |
| 503 | Service Unavailable | Temporary service outage |

### Common Error Codes

| Error Code | Description | Solution |
|------------|-------------|----------|
| `INVALID_API_KEY` | API key is missing or invalid | Check API key configuration |
| `RATE_LIMIT_EXCEEDED` | Too many requests | Wait for rate limit reset |
| `INVALID_COIN_ID` | Coin ID not found | Check coin ID spelling |
| `INVALID_TIMEFRAME` | Invalid timeframe parameter | Use: minute, hour, day, week, month |
| `INVALID_DATE_RANGE` | Invalid date range | Check date format and range |
| `DATABASE_ERROR` | Database connection issue | Temporary - retry request |

## Pagination

For endpoints that return large datasets, use pagination:

```bash
# Get first page (default)
GET /coins?limit=100&offset=0

# Get second page
GET /coins?limit=100&offset=100

# Get third page
GET /coins?limit=100&offset=200
```

**Pagination Response**:
```json
{
  "meta": {
    "total_count": 1000,
    "page": 1,
    "limit": 100,
    "total_pages": 10,
    "has_next_page": true,
    "has_previous_page": false
  }
}
```

## Best Practices

### 1. Error Handling
Always check the `success` field and handle errors appropriately:

```javascript
const response = await fetch(url, { headers: { 'X-API-Key': apiKey } });
const data = await response.json();

if (!data.success) {
  console.error('API Error:', data.errors);
  return;
}

// Process data.data
```

### 2. Rate Limiting
Monitor rate limit headers and implement backoff:

```javascript
const rateLimitRemaining = response.headers.get('RateLimit-Remaining');
if (rateLimitRemaining < 10) {
  // Slow down requests
  await new Promise(resolve => setTimeout(resolve, 1000));
}
```

### 3. Caching
Cache responses to reduce API calls:

```javascript
// Cache market data for 1 minute
const cacheKey = `market_summary_${Date.now() / 60000 | 0}`;
```

### 4. Timeouts
Set appropriate timeouts for requests:

```javascript
const controller = new AbortController();
setTimeout(() => controller.abort(), 10000); // 10 second timeout

fetch(url, { 
  signal: controller.signal,
  headers: { 'X-API-Key': apiKey }
});
```

## SDKs and Examples

### JavaScript/Node.js Example

```javascript
class PulseGuardAPI {
  constructor(apiKey, baseURL = 'https://your-server.com/api/v1') {
    this.apiKey = apiKey;
    this.baseURL = baseURL;
  }

  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const response = await fetch(url, {
      ...options,
      headers: {
        'X-API-Key': this.apiKey,
        'Content-Type': 'application/json',
        ...options.headers
      }
    });

    const data = await response.json();
    
    if (!data.success) {
      throw new Error(data.errors?.[0]?.message || 'API request failed');
    }
    
    return data.data;
  }

  // Get system status
  async getSystemStatus() {
    return this.request('/system/status');
  }

  // Get top cryptocurrencies
  async getTopCoins(count = 10) {
    return this.request(`/market/top/${count}`);
  }

  // Get coin price history
  async getCoinHistory(coinId, timeframe = 'hour', limit = 24) {
    return this.request(`/coins/${coinId}/history?timeframe=${timeframe}&limit=${limit}`);
  }
}

// Usage
const api = new PulseGuardAPI('your-api-key');
const topCoins = await api.getTopCoins(10);
```

### Python Example

```python
import requests
from typing import Dict, Any, Optional

class PulseGuardAPI:
    def __init__(self, api_key: str, base_url: str = "https://your-server.com/api/v1"):
        self.api_key = api_key
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            'X-API-Key': api_key,
            'Content-Type': 'application/json'
        })

    def request(self, endpoint: str, params: Optional[Dict] = None) -> Dict[str, Any]:
        url = f"{self.base_url}{endpoint}"
        response = self.session.get(url, params=params)
        data = response.json()
        
        if not data.get('success'):
            raise Exception(data.get('errors', [{}])[0].get('message', 'API request failed'))
        
        return data['data']

    def get_system_status(self) -> Dict[str, Any]:
        return self.request('/system/status')

    def get_top_coins(self, count: int = 10) -> List[Dict[str, Any]]:
        return self.request(f'/market/top/{count}')

    def get_coin_history(self, coin_id: str, timeframe: str = 'hour', limit: int = 24) -> List[Dict[str, Any]]:
        params = {'timeframe': timeframe, 'limit': limit}
        return self.request(f'/coins/{coin_id}/history', params)

# Usage
api = PulseGuardAPI('your-api-key')
top_coins = api.get_top_coins(10)
```

## Support

For API support and questions:
- **Documentation**: Check this documentation first
- **Status Page**: Monitor API status at your status page
- **Rate Limits**: Monitor your usage in API responses
- **Issues**: Report bugs in your issue tracker
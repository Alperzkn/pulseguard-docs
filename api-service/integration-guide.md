# PulseGuard API Integration Guide for Developers

## Overview

This guide explains how to integrate your application with the PulseGuard cryptocurrency data API. The API provides real-time and historical cryptocurrency price data with market cap rankings.

## API Base Information

- **Base URL**: `http://your-droplet-ip:3000/api/v1`
- **Authentication**: API Key (header-based)
- **Response Format**: JSON
- **Rate Limits**: 100 requests per 15-minute window

## Quick Reference

| Endpoint Category | Purpose |
|------------------|---------|
| **Coins** (`/coins`) | Individual cryptocurrency data and historical prices |
| **Market** (`/market`) | Market rankings, summaries, and multi-coin data |
| **API Usage** (`/api`) | CoinGecko API usage monitoring and cost optimization |
| **System** (`/system`) | Health checks, data completeness, and system monitoring |

## Authentication Setup

### 1. API Key Configuration

Store your API key securely in your application's environment variables:

```bash
# .env file in your application
PULSEGUARD_API_KEY=your-secret-api-key-here
PULSEGUARD_API_BASE_URL=http://your-droplet-ip:3000/api/v1
```

### 2. Request Headers

Include the API key in every request:

```javascript
const headers = {
  'X-API-Key': process.env.PULSEGUARD_API_KEY,
  'Content-Type': 'application/json'
};
```

## API Endpoints Reference

### 📊 **Coins Endpoints**

#### 1. Get All Cryptocurrencies

**GET** `/coins`

Get a paginated list of all available cryptocurrencies with basic information.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/coins?limit=50&offset=0&sort_by=rank"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/coins?limit=50&offset=0`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Query Parameters:**
- `limit` (optional): Number of coins to return (default: 50, max: 1000)
- `offset` (optional): Number of coins to skip for pagination (default: 0)
- `sort_by` (optional): Sort by 'rank' or 'name' (default: 'rank')

**Response Example:**
```json
{
  "success": true,
  "data": [
    {
      "coin_id": "bitcoin",
      "rank": 1,
      "market_cap_usd": "865432100000.00",
      "trading_volume_24h": "28765432100.00",
      "current_price": "45123.45",
      "latest_minute_price": "45125.67",
      "latest_price_timestamp": "2025-01-15T10:30:00Z",
      "last_updated": "2025-01-15T10:29:45Z"
    }
  ],
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z",
    "total_count": 1000,
    "page": 1,
    "per_page": 50,
    "total_pages": 20
  }
}
```

#### 2. Get Specific Cryptocurrency Details

**GET** `/coins/{coin_id}`

Get detailed information for a specific cryptocurrency including current price, market cap, and ranking.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/coins/bitcoin"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/coins/bitcoin`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Response Example:**
```json
{
  "success": true,
  "data": {
    "coin_id": "bitcoin",
    "rank": 1,
    "market_cap_usd": "865432100000.00",
    "trading_volume_24h": "28765432100.00",
    "current_price": "45123.45",
    "latest_minute_price": "45125.67",
    "latest_price_timestamp": "2025-01-15T10:30:00Z",
    "latest_hourly_price": "45098.23",
    "latest_hourly_timestamp": "2025-01-15T10:00:00Z",
    "last_updated": "2025-01-15T10:29:45Z"
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  }
}
```

#### 3. Get Current Price

**GET** `/coins/{coin_id}/price`

Get the most recent price for a specific cryptocurrency in the specified timeframe.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/coins/bitcoin/price?timeframe=minute"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/coins/bitcoin/price?timeframe=minute`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Query Parameters:**
- `timeframe` (optional): 'minute', 'hour', 'day', 'week', 'month' (default: 'minute')

**Response Example:**
```json
{
  "success": true,
  "data": {
    "coin_id": "bitcoin",
    "timeframe": "minute",
    "price": "45125.67",
    "timestamp": "2025-01-15T10:30:00Z",
    "rank": 1
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  }
}
```

#### 4. Get Historical Price Data

**GET** `/coins/{coin_id}/history`

Get historical price data for a specific cryptocurrency with flexible filtering options.

```bash
# Get last 24 hours of Bitcoin prices
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/coins/bitcoin/history?timeframe=hour&limit=24"

# Get Bitcoin prices for specific date range
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/coins/bitcoin/history?timeframe=day&from=2025-01-01&to=2025-01-15"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/coins/bitcoin/history?timeframe=hour&limit=24`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Query Parameters:**
- `timeframe` (required): 'minute', 'hour', 'day', 'week', 'month'
- `limit` (optional): Max number of data points (default: 100, max: 10000)
- `from` (optional): Start timestamp (ISO 8601 format)
- `to` (optional): End timestamp (ISO 8601 format)

**Response Example:**
```json
{
  "success": true,
  "data": {
    "coin_id": "bitcoin",
    "timeframe": "hour",
    "history": [
      {
        "price": "45125.67",
        "timestamp": "2025-01-15T10:00:00Z",
        "rank": 1
      },
      {
        "price": "45098.23",
        "timestamp": "2025-01-15T09:00:00Z",
        "rank": 1
      }
    ],
    "count": 24
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z",
    "query_params": {
      "timeframe": "hour",
      "limit": 24,
      "from": null,
      "to": null
    }
  }
}
```

### 🏪 **Market Endpoints**

#### 5. Get Market Rankings

**GET** `/market/rankings`

Get cryptocurrencies ranked by market capitalization with pagination support.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/market/rankings?limit=10&offset=0"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/market/rankings?limit=10`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Query Parameters:**
- `limit` (optional): Number of coins to return (default: 50, max: 1000)
- `offset` (optional): Number of coins to skip for pagination (default: 0)

**Response Example:**
```json
{
  "success": true,
  "data": [
    {
      "coin_id": "bitcoin",
      "rank": 1,
      "market_cap_usd": "865432100000.00",
      "trading_volume_24h": "28765432100.00",
      "current_price": "45123.45",
      "latest_minute_price": "45125.67",
      "latest_price_timestamp": "2025-01-15T10:30:00Z",
      "last_updated": "2025-01-15T10:29:45Z"
    }
  ],
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z",
    "total_count": 1000,
    "page": 1,
    "per_page": 10,
    "total_pages": 100
  }
}
```

#### 6. Get Top N Cryptocurrencies

**GET** `/market/top/{count}`

Get the top N cryptocurrencies by market capitalization with additional metrics like 24h price change.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/market/top/10"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/market/top/10`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Path Parameters:**
- `count` (required): Number of top coins to return (1-1000)

**Response Example:**
```json
{
  "success": true,
  "data": [
    {
      "coin_id": "bitcoin",
      "rank": 1,
      "market_cap_usd": "865432100000.00",
      "trading_volume_24h": "28765432100.00",
      "current_price": "45123.45",
      "latest_minute_price": "45125.67",
      "latest_price_timestamp": "2025-01-15T10:30:00Z",
      "price_change_24h_percent": "2.45",
      "last_updated": "2025-01-15T10:29:45Z"
    }
  ],
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z",
    "requested_count": 10,
    "returned_count": 10
  }
}
```

#### 7. Get Market Summary

**GET** `/market/summary`

Get overall market statistics including total market cap and volume.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/market/summary"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/market/summary`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Response Example:**
```json
{
  "success": true,
  "data": {
    "total_coins": 1000,
    "total_market_cap": "2456789123456.78",
    "total_volume_24h": "89123456789.12",
    "avg_market_cap": "2456789123.46",
    "last_data_update": "2025-01-15T10:29:45Z"
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  }
}
```

#### 8. Search Cryptocurrencies

**GET** `/market/search`

Search for cryptocurrencies by name or ID with intelligent ranking.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/market/search?q=bitcoin&limit=20"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/market/search?q=bitcoin&limit=10`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Query Parameters:**
- `q` (required): Search query string
- `limit` (optional): Max results to return (default: 20, max: 100)

**Response Example:**
```json
{
  "success": true,
  "data": {
    "query": "bitcoin",
    "results": [
      {
        "coin_id": "bitcoin",
        "rank": 1,
        "market_cap_usd": "865432100000.00",
        "trading_volume_24h": "28765432100.00",
        "current_price": "45123.45",
        "last_updated": "2025-01-15T10:29:45Z"
      }
    ],
    "count": 1
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  }
}
```

#### 9. Get Coins by Rank Range

**GET** `/market/range`

Get cryptocurrencies within a specific market cap rank range.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/market/range?min_rank=10&max_rank=20"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/market/range?min_rank=10&max_rank=20`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Query Parameters:**
- `min_rank` (required): Minimum rank (1-10000)
- `max_rank` (required): Maximum rank (1-10000)

**Response Example:**
```json
{
  "success": true,
  "data": {
    "range": {
      "min_rank": 10,
      "max_rank": 20
    },
    "coins": [
      {
        "coin_id": "chainlink",
        "rank": 10,
        "market_cap_usd": "8765432100.00",
        "trading_volume_24h": "876543210.00",
        "current_price": "15.67",
        "latest_minute_price": "15.68",
        "latest_price_timestamp": "2025-01-15T10:30:00Z",
        "last_updated": "2025-01-15T10:29:45Z"
      }
    ],
    "count": 11
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  }
}
```

#### 10. Get Multi-Coin Historical Data

**GET** `/market/history`

Get historical data for multiple cryptocurrencies simultaneously.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/market/history?coin_ids=bitcoin,ethereum,cardano&timeframe=hour&limit=24"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/market/history?coin_ids=bitcoin,ethereum&timeframe=hour&limit=24`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Query Parameters:**
- `coin_ids` (required): Comma-separated list of coin IDs (max 100)
- `timeframe` (required): 'minute', 'hour', 'day', 'week', 'month'
- `limit` (optional): Max data points per coin (default: 100, max: 10000)
- `from` (optional): Start timestamp (ISO 8601)
- `to` (optional): End timestamp (ISO 8601)

**Response Example:**
```json
{
  "success": true,
  "data": {
    "coins": ["bitcoin", "ethereum"],
    "timeframe": "hour",
    "history": {
      "bitcoin": [
        {
          "price": "45125.67",
          "timestamp": "2025-01-15T10:00:00Z",
          "rank": 1
        }
      ],
      "ethereum": [
        {
          "price": "3456.78",
          "timestamp": "2025-01-15T10:00:00Z",
          "rank": 2
        }
      ]
    },
    "coin_count": 2
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z",
    "query_params": {
      "timeframe": "hour",
      "limit": 24,
      "from": null,
      "to": null
    }
  }
}
```

### 📈 **API Usage Endpoints**

#### 11. Get CoinGecko API Usage Statistics

**GET** `/api/coingecko-usage`

Get real-time CoinGecko API usage statistics and recommendations based on the actual data collection system configuration. This endpoint provides accurate calculations of API usage patterns and upgrade recommendations.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/api/coingecko-usage"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/api/coingecko-usage`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Response Example:**
```json
{
  "success": true,
  "data": {
    "current_plan": "Demo",
    "usage": {
      "last_minute": {
        "calls": 4,
        "limit": 30,
        "percentage": 13.33
      },
      "last_hour": {
        "calls": 240,
        "limit": 1800,
        "percentage": 13.33
      },
      "last_day": {
        "calls": 5760,
        "limit": 43200,
        "percentage": 13.33
      },
      "last_month": {
        "calls": 172800,
        "limit": 10000,
        "percentage": 1728
      }
    },
    "calculation_details": {
      "total_coins": 1000,
      "coins_per_batch": 250,
      "api_calls_per_collection": 4,
      "collections_per_minute": 1,
      "batch_delay_seconds": 6,
      "collection_strategy": "Every minute via cron, batched API calls to /simple/price"
    },
    "upgrade_recommendations": [
      {
        "plan": "Demo",
        "price": "Free",
        "limit": "30 calls/min, 10K/month",
        "status": "current",
        "recommended": false
      },
      {
        "plan": "Analyst",
        "price": "$129/month",
        "limit": "500 calls/min, 500K/month",
        "status": "available",
        "recommended": true
      },
      {
        "plan": "Enterprise",
        "price": "Custom",
        "limit": "Custom limits",
        "status": "enterprise",
        "recommended": false
      }
    ]
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z",
    "calculated_from": "real_system_data",
    "data_source": "system_status"
  },
  "errors": null
}
```

**Key Features:**
- **Real-time calculations** based on actual collection system data
- **Accurate usage metrics** using batched API calls (250 coins per request)
- **Upgrade recommendations** with current 2025 CoinGecko pricing
- **Collection strategy details** showing how data is collected every minute
- **Monthly usage projections** based on actual collection patterns

**Use Cases:**
- Dashboard monitoring of API usage costs
- Planning for CoinGecko API plan upgrades
- Understanding data collection efficiency
- Cost optimization for cryptocurrency data collection

### 🔧 **System Endpoints**

#### 12. Health Check

**GET** `/system/health` (No Authentication Required)

Simple health check endpoint for monitoring API availability.

```bash
curl "http://your-droplet-ip:3000/api/v1/system/health"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/system/health`);
const data = await response.json();
```

**Response Example:**
```json
{
  "success": true,
  "data": {
    "api": {
      "status": "operational",
      "uptime": 86400,
      "memory": {
        "rss": 52428800,
        "heapTotal": 20971520,
        "heapUsed": 15728640
      },
      "version": "v1"
    },
    "database": {
      "status": "connected",
      "current_time": "2025-01-15T10:30:15Z",
      "postgres_version": "PostgreSQL 13.7 on x86_64"
    },
    "timestamp": "2025-01-15T10:30:15Z"
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  }
}
```

#### 12. System Status

**GET** `/system/status`

Get comprehensive system health including data collection status and freshness.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/system/status"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/system/status`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Response Example:**
```json
{
  "success": true,
  "data": {
    "system_health": "operational",
    "latest_collection": {
      "collection_date": "2025-01-15",
      "total_coins_attempted": 1000,
      "total_coins_successful": 998,
      "collection_mode": "top1000",
      "started_at": "2025-01-15T10:00:00Z",
      "completed_at": "2025-01-15T10:05:30Z",
      "duration_seconds": 330
    },
    "data_freshness": {
      "minute": {
        "latest_timestamp": "2025-01-15T10:30:00Z",
        "total_records": 998,
        "unique_coins": 998
      },
      "hourly": {
        "latest_timestamp": "2025-01-15T10:00:00Z",
        "total_records": 998,
        "unique_coins": 998
      }
    },
    "coin_tracking": {
      "total_coins_tracked": 1000,
      "priorities_last_updated": "2025-01-15T06:00:00Z",
      "top_100_coins": 100,
      "top_1000_coins": 1000
    },
    "api_status": {
      "uptime": 86400,
      "memory_usage": {
        "rss": 52428800,
        "heapTotal": 20971520,
        "heapUsed": 15728640
      },
      "node_version": "v18.19.0"
    },
    "timestamp": "2025-01-15T10:30:15Z"
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  }
}
```

#### 13. Collection History

**GET** `/system/collections`

Get historical data collection statistics and performance metrics.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/system/collections?limit=30"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/system/collections?limit=30`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});

const data = await response.json();
```

**Query Parameters:**
- `limit` (optional): Number of records to return (default: 30, max: 365)
- `from` (optional): Start date (YYYY-MM-DD)
- `to` (optional): End date (YYYY-MM-DD)

#### 14. Database Statistics

**GET** `/system/database`

Get database performance metrics and table statistics.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/system/database"
```

#### 15. Test Database Connection

**POST** `/system/test-db`

Test database connectivity and return connection details.

```bash
curl -H "X-API-Key: your-api-key" \
  -X POST "http://your-droplet-ip:3000/api/v1/system/test-db"
```

#### 16. Data Completeness Metrics

**GET** `/system/completeness`

Get comprehensive data completeness metrics for monitoring missing data across different timeframes.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/system/completeness"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/system/completeness`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});
const data = await response.json();
```

**Response Example:**
```json
{
  "success": true,
  "data": {
    "timeframes": {
      "last_minute": {
        "expected": 1000,
        "actual": 1000,
        "missing": 0,
        "success_rate": 100,
        "last_updated": "2025-01-15T10:30:00Z"
      },
      "last_hour": {
        "expected": 1000,
        "actual": 1000,
        "missing": 0,
        "success_rate": 100,
        "last_updated": "2025-01-15T10:00:00Z"
      },
      "last_day": {
        "expected": 1000,
        "actual": 1000,
        "missing": 0,
        "success_rate": 100,
        "last_updated": "2025-01-15T00:00:00Z"
      }
    },
    "tables": {
      "minute_close": {
        "current_period_count": 1000,
        "expected_count": 1000,
        "success_rate": 100,
        "status": "healthy"
      },
      "hourly_close": {
        "current_period_count": 1000,
        "expected_count": 1000,
        "success_rate": 100,
        "status": "healthy"
      },
      "daily_close": {
        "current_period_count": 1000,
        "expected_count": 1000,
        "success_rate": 100,
        "status": "healthy"
      }
    },
    "alerts": []
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  },
  "errors": null
}
```

#### 17. Missing Coins List

**GET** `/system/missing-coins`

Get a list of cryptocurrency coins that are missing from the latest data collection.

```bash
curl -H "X-API-Key: your-api-key" \
  "http://your-droplet-ip:3000/api/v1/system/missing-coins"
```

```javascript
const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/system/missing-coins`, {
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  }
});
const data = await response.json();
```

**Response Example:**
```json
{
  "success": true,
  "data": [],
  "meta": {
    "timestamp": "2025-01-15T10:30:15Z"
  },
  "errors": null
}
```

**Use Cases:**
- Dashboard monitoring for data completeness
- Alert systems for missing cryptocurrency data
- Collection process troubleshooting
- Performance metrics tracking

## Error Handling

### HTTP Status Codes

- `200` - Success
- `400` - Bad Request (invalid parameters)
- `401` - Unauthorized (invalid/missing API key)
- `404` - Not Found (coin doesn't exist)
- `429` - Too Many Requests (rate limit exceeded)
- `500` - Internal Server Error

### Error Response Format

```json
{
  "success": false,
  "data": null,
  "errors": ["Invalid API key provided"],
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

### Error Handling in Code

```javascript
async function fetchCoinPrice(coinId) {
  try {
    const response = await fetch(`${process.env.PULSEGUARD_API_BASE_URL}/coins/${coinId}/price`, {
      headers: {
        'X-API-Key': process.env.PULSEGUARD_API_KEY
      }
    });

    if (!response.ok) {
      if (response.status === 401) {
        throw new Error('Invalid API key');
      } else if (response.status === 429) {
        throw new Error('Rate limit exceeded. Please wait and try again.');
      } else if (response.status === 404) {
        throw new Error(`Cryptocurrency '${coinId}' not found`);
      }
      throw new Error(`API request failed: ${response.status}`);
    }

    const data = await response.json();
    
    if (!data.success) {
      throw new Error(data.errors?.[0] || 'API request failed');
    }

    return data.data;
  } catch (error) {
    console.error('Failed to fetch coin price:', error.message);
    throw error;
  }
}
```

## Rate Limiting

### Current Limits
- **100 requests per 15-minute window** per API key
- Rate limits reset every 15 minutes

### Rate Limit Headers
The API includes rate limit information in response headers:

```javascript
const response = await fetch(url, { headers });

console.log('Rate limit remaining:', response.headers.get('X-RateLimit-Remaining'));
console.log('Rate limit reset time:', response.headers.get('X-RateLimit-Reset'));
```

### Handling Rate Limits

```javascript
async function fetchWithRateLimit(url, options) {
  const response = await fetch(url, options);
  
  if (response.status === 429) {
    const resetTime = response.headers.get('X-RateLimit-Reset');
    const waitTime = new Date(resetTime) - new Date();
    
    console.log(`Rate limited. Waiting ${waitTime}ms before retry...`);
    await new Promise(resolve => setTimeout(resolve, waitTime));
    
    // Retry request
    return fetchWithRateLimit(url, options);
  }
  
  return response;
}
```

## Integration Examples

### React.js Example

```javascript
import { useState, useEffect } from 'react';

function CoinPrice({ coinId }) {
  const [price, setPrice] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchPrice = async () => {
      try {
        setLoading(true);
        const response = await fetch(`${process.env.REACT_APP_PULSEGUARD_API_BASE_URL}/coins/${coinId}/price`, {
          headers: {
            'X-API-Key': process.env.REACT_APP_PULSEGUARD_API_KEY
          }
        });

        if (!response.ok) {
          throw new Error(`Failed to fetch price: ${response.status}`);
        }

        const data = await response.json();
        setPrice(data.data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchPrice();
    const interval = setInterval(fetchPrice, 60000); // Update every minute

    return () => clearInterval(interval);
  }, [coinId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h3>{coinId.toUpperCase()}</h3>
      <p>Price: ${price?.price}</p>
      <p>Updated: {new Date(price?.timestamp).toLocaleString()}</p>
    </div>
  );
}
```

### Node.js/Express Backend Example

```javascript
const express = require('express');
const axios = require('axios');

const app = express();

// API client configuration
const apiClient = axios.create({
  baseURL: process.env.PULSEGUARD_API_BASE_URL,
  headers: {
    'X-API-Key': process.env.PULSEGUARD_API_KEY
  },
  timeout: 10000
});

// Endpoint to get top cryptocurrencies for your app
app.get('/api/top-cryptos', async (req, res) => {
  try {
    const { count = 10 } = req.query;
    const response = await apiClient.get(`/market/top/${count}`);
    
    res.json({
      success: true,
      data: response.data.data
    });
  } catch (error) {
    console.error('Failed to fetch top cryptos:', error.message);
    res.status(500).json({
      success: false,
      error: 'Failed to fetch cryptocurrency data'
    });
  }
});

app.listen(3001, () => {
  console.log('App server running on port 3001');
});
```

## Best Practices

### 1. Caching
Cache API responses to reduce requests and improve performance:

```javascript
const cache = new Map();
const CACHE_DURATION = 60000; // 1 minute

async function getCachedPrice(coinId) {
  const cacheKey = `price:${coinId}`;
  const cached = cache.get(cacheKey);
  
  if (cached && (Date.now() - cached.timestamp) < CACHE_DURATION) {
    return cached.data;
  }
  
  const fresh = await fetchCoinPrice(coinId);
  cache.set(cacheKey, {
    data: fresh,
    timestamp: Date.now()
  });
  
  return fresh;
}
```

### 2. Error Recovery
Implement retry logic with exponential backoff:

```javascript
async function fetchWithRetry(url, options, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      if (response.ok) return response;
      
      if (response.status === 429 && attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      
      throw new Error(`Request failed: ${response.status}`);
    } catch (error) {
      if (attempt === maxRetries) throw error;
      
      const delay = Math.pow(2, attempt) * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### 3. Environment Variables
Always use environment variables for configuration:

```javascript
// Validate required environment variables at startup
const requiredEnvVars = [
  'PULSEGUARD_API_KEY',
  'PULSEGUARD_API_BASE_URL'
];

requiredEnvVars.forEach(varName => {
  if (!process.env[varName]) {
    throw new Error(`Missing required environment variable: ${varName}`);
  }
});
```

## Testing

### Testing API Integration

```javascript
// Jest test example
describe('PulseGuard API Integration', () => {
  test('should fetch Bitcoin price', async () => {
    const price = await fetchCoinPrice('bitcoin');
    
    expect(price).toHaveProperty('price');
    expect(price).toHaveProperty('timestamp');
    expect(typeof price.price).toBe('number');
    expect(price.price).toBeGreaterThan(0);
  });

  test('should handle invalid coin ID', async () => {
    await expect(fetchCoinPrice('invalid-coin')).rejects.toThrow('not found');
  });
});
```

## Security Considerations

1. **Never expose API keys in client-side code**
2. **Use HTTPS in production**
3. **Validate all API responses**
4. **Implement proper error handling**
5. **Set reasonable request timeouts**

## Support

If you encounter issues:
1. Check API key validity
2. Verify network connectivity to the droplet
3. Check rate limits
4. Review error messages for specific guidance

For additional help, contact the development team with specific error messages and request details.
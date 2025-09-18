# API Service Development Guide

## Development Environment Setup

### Prerequisites

- Node.js 18+ installed
- PostgreSQL database running
- Access to PulseGuard data collector database
- Code editor (VS Code recommended)

### Local Development Setup

1. **Clone and Install**
   ```bash
   git clone your-api-repo pulseguard-api
   cd pulseguard-api
   npm install
   ```

2. **Environment Configuration**
   ```bash
   cp .env.api.example .env.api
   ```

   Configure `.env.api`:
   ```bash
   # Database (same as data collector)
   DATABASE_URL=postgresql://pulseguard_user:password@localhost:5432/crypto_db
   
   # Server Configuration
   PORT=3000
   NODE_ENV=development
   
   # Authentication
   API_KEY_SECRET=your-development-api-key
   
   # Rate Limiting
   RATE_LIMIT_MAX_REQUESTS=200
   RATE_LIMIT_WINDOW_MS=900000
   
   # CORS
   CORS_ORIGIN=*
   
   # Logging
   LOG_LEVEL=debug
   ```

3. **Start Development Server**
   ```bash
   # With auto-reload
   npm run dev
   
   # Or normal start
   npm start
   ```

4. **Verify Setup**
   ```bash
   curl http://localhost:3000/health
   # Should return: {"status":"healthy","timestamp":"..."}
   ```

## Project Structure

```
src/
├── server.js              # Express application entry point
├── config/
│   └── database.js         # Database connection configuration
├── controllers/            # Request handlers
│   ├── CoinsController.js  # Coin-related endpoints
│   ├── MarketController.js # Market data endpoints
│   ├── SystemController.js # System monitoring endpoints
│   └── ApiController.js    # API usage tracking
├── models/                 # Database interaction layer
│   ├── CoinsModel.js       # Coin data queries
│   ├── MarketModel.js      # Market data queries
│   ├── SystemModel.js      # System status queries
│   └── ApiModel.js         # API usage tracking
├── middleware/             # Express middleware
│   ├── auth.js            # API key authentication
│   ├── rateLimiter.js     # Request rate limiting
│   ├── validation.js      # Input validation
│   └── errorHandler.js    # Error handling
├── routes/                # Route definitions
│   ├── coins.js           # /api/v1/coins/* routes
│   ├── market.js          # /api/v1/market/* routes
│   ├── system.js          # /api/v1/system/* routes
│   └── api.js             # /api/v1/api/* routes
└── utils/                 # Utility functions
    ├── logger.js          # Logging configuration
    ├── cache.js           # Response caching
    └── helpers.js         # Common helper functions
```

## Development Workflow

### 1. Adding New Endpoints

#### Step 1: Define Route
```javascript
// src/routes/coins.js
const express = require('express');
const router = express.Router();
const CoinsController = require('../controllers/CoinsController');

router.get('/trending', CoinsController.getTrendingCoins);

module.exports = router;
```

#### Step 2: Create Controller
```javascript
// src/controllers/CoinsController.js
const CoinsModel = require('../models/CoinsModel');

class CoinsController {
  static async getTrendingCoins(req, res, next) {
    try {
      const { limit = 10, timeframe = '24h' } = req.query;
      
      const trendingCoins = await CoinsModel.getTrendingCoins({
        limit: parseInt(limit),
        timeframe
      });

      res.json({
        success: true,
        data: trendingCoins,
        meta: {
          timestamp: new Date().toISOString(),
          total_count: trendingCoins.length
        }
      });
    } catch (error) {
      next(error);
    }
  }
}

module.exports = CoinsController;
```

#### Step 3: Create Model Method
```javascript
// src/models/CoinsModel.js
const pool = require('../config/database');

class CoinsModel {
  static async getTrendingCoins({ limit, timeframe }) {
    const query = `
      SELECT 
        coin_id,
        symbol,
        name,
        current_price,
        price_change_percentage_24h,
        volume_24h,
        market_cap_rank
      FROM coin_priorities cp
      JOIN LATERAL (
        SELECT price_usd as current_price
        FROM minute_close mc
        WHERE mc.coin_id = cp.coin_id
        ORDER BY minute_ts DESC
        LIMIT 1
      ) latest_price ON true
      WHERE cp.market_cap_rank <= $1
      ORDER BY price_change_percentage_24h DESC
      LIMIT $2;
    `;
    
    const result = await pool.query(query, [1000, limit]);
    return result.rows;
  }
}

module.exports = CoinsModel;
```

#### Step 4: Add Tests
```javascript
// tests/coins.test.js
const request = require('supertest');
const app = require('../src/server');

describe('GET /api/v1/coins/trending', () => {
  test('should return trending coins', async () => {
    const response = await request(app)
      .get('/api/v1/coins/trending')
      .set('X-API-Key', process.env.API_KEY_SECRET)
      .expect(200);

    expect(response.body.success).toBe(true);
    expect(Array.isArray(response.body.data)).toBe(true);
    expect(response.body.data.length).toBeLessThanOrEqual(10);
  });
});
```

### 2. Database Query Patterns

#### Time-Series Queries
```javascript
// Get recent price data with pagination
static async getPriceHistory(coinId, { timeframe, limit, offset }) {
  const tableName = this.getTableName(timeframe);
  const timeColumn = this.getTimeColumn(timeframe);
  
  const query = `
    SELECT 
      ${timeColumn} as timestamp,
      price_usd,
      volume_24h_usd,
      market_cap_usd
    FROM ${tableName}
    WHERE coin_id = $1
    ORDER BY ${timeColumn} DESC
    LIMIT $2 OFFSET $3;
  `;
  
  const result = await pool.query(query, [coinId, limit, offset]);
  return result.rows;
}

// Aggregate data efficiently
static async getMarketSummary() {
  const query = `
    WITH latest_data AS (
      SELECT DISTINCT ON (coin_id)
        coin_id,
        price_usd,
        market_cap_usd,
        volume_24h_usd
      FROM minute_close
      WHERE minute_ts > NOW() - INTERVAL '1 hour'
      ORDER BY coin_id, minute_ts DESC
    )
    SELECT 
      COUNT(*) as active_cryptocurrencies,
      SUM(market_cap_usd) as total_market_cap,
      SUM(volume_24h_usd) as total_volume_24h,
      AVG(price_usd) as average_price
    FROM latest_data;
  `;
  
  const result = await pool.query(query);
  return result.rows[0];
}
```

#### Performance Optimizations
```javascript
// Use prepared statements for repeated queries
static async prepareStatements() {
  await pool.query(`
    PREPARE get_coin_price (text, timestamptz) AS
    SELECT price_usd FROM minute_close 
    WHERE coin_id = $1 AND minute_ts <= $2 
    ORDER BY minute_ts DESC LIMIT 1;
  `);
}

// Use connection pooling efficiently
static async batchQuery(queries) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const results = await Promise.all(
      queries.map(query => client.query(query.text, query.params))
    );
    await client.query('COMMIT');
    return results;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

### 3. Authentication & Security

#### API Key Middleware
```javascript
// src/middleware/auth.js
const authMiddleware = (req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  
  if (!apiKey) {
    return res.status(401).json({
      success: false,
      errors: [{ code: 'MISSING_API_KEY', message: 'API key is required' }]
    });
  }
  
  if (apiKey !== process.env.API_KEY_SECRET) {
    return res.status(401).json({
      success: false,
      errors: [{ code: 'INVALID_API_KEY', message: 'Invalid API key' }]
    });
  }
  
  next();
};

module.exports = authMiddleware;
```

#### Input Validation
```javascript
// src/middleware/validation.js
const { body, query, validationResult } = require('express-validator');

const validatePriceHistoryQuery = [
  query('timeframe').isIn(['minute', 'hour', 'day', 'week', 'month']),
  query('limit').optional().isInt({ min: 1, max: 1000 }),
  query('offset').optional().isInt({ min: 0 }),
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        success: false,
        errors: errors.array().map(err => ({
          code: 'VALIDATION_ERROR',
          message: err.msg,
          field: err.param
        }))
      });
    }
    next();
  }
];

module.exports = { validatePriceHistoryQuery };
```

### 4. Error Handling

#### Global Error Handler
```javascript
// src/middleware/errorHandler.js
const logger = require('../utils/logger');

const errorHandler = (error, req, res, next) => {
  logger.error('API Error:', {
    error: error.message,
    stack: error.stack,
    url: req.url,
    method: req.method,
    ip: req.ip
  });

  // Database connection errors
  if (error.code === 'ECONNREFUSED' || error.code === '57P01') {
    return res.status(503).json({
      success: false,
      errors: [{ 
        code: 'DATABASE_UNAVAILABLE', 
        message: 'Database temporarily unavailable' 
      }]
    });
  }

  // PostgreSQL errors
  if (error.code && error.code.startsWith('23')) {
    return res.status(400).json({
      success: false,
      errors: [{ 
        code: 'DATABASE_CONSTRAINT_ERROR', 
        message: 'Invalid data provided' 
      }]
    });
  }

  // Default server error
  res.status(500).json({
    success: false,
    errors: [{ 
      code: 'INTERNAL_SERVER_ERROR', 
      message: 'An unexpected error occurred' 
    }]
  });
};

module.exports = errorHandler;
```

### 5. Testing

#### Test Setup
```javascript
// tests/setup.js
const { Pool } = require('pg');

// Test database configuration
const testDbConfig = {
  connectionString: process.env.TEST_DATABASE_URL,
  max: 5
};

const testPool = new Pool(testDbConfig);

beforeAll(async () => {
  // Create test data
  await testPool.query(`
    INSERT INTO coin_priorities (coin_id, market_cap_rank, symbol, name)
    VALUES ('bitcoin', 1, 'BTC', 'Bitcoin'), ('ethereum', 2, 'ETH', 'Ethereum');
  `);
});

afterAll(async () => {
  // Cleanup test data
  await testPool.query('TRUNCATE coin_priorities CASCADE;');
  await testPool.end();
});
```

#### Integration Tests
```javascript
// tests/integration/api.test.js
const request = require('supertest');
const app = require('../../src/server');

describe('API Integration Tests', () => {
  const apiKey = process.env.API_KEY_SECRET;

  test('GET /api/v1/system/status', async () => {
    const response = await request(app)
      .get('/api/v1/system/status')
      .set('X-API-Key', apiKey)
      .expect(200);

    expect(response.body.success).toBe(true);
    expect(response.body.data).toHaveProperty('status');
    expect(response.body.data).toHaveProperty('uptime');
  });

  test('GET /api/v1/coins without auth should fail', async () => {
    await request(app)
      .get('/api/v1/coins')
      .expect(401);
  });

  test('GET /api/v1/market/top/10', async () => {
    const response = await request(app)
      .get('/api/v1/market/top/10')
      .set('X-API-Key', apiKey)
      .expect(200);

    expect(response.body.success).toBe(true);
    expect(Array.isArray(response.body.data)).toBe(true);
    expect(response.body.data.length).toBeLessThanOrEqual(10);
  });
});
```

#### Unit Tests
```javascript
// tests/unit/models/CoinsModel.test.js
const CoinsModel = require('../../../src/models/CoinsModel');
const pool = require('../../../src/config/database');

jest.mock('../../../src/config/database');

describe('CoinsModel', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('getAllCoins should return formatted coin data', async () => {
    const mockData = [
      { coin_id: 'bitcoin', symbol: 'BTC', name: 'Bitcoin' }
    ];
    
    pool.query.mockResolvedValue({ rows: mockData });

    const result = await CoinsModel.getAllCoins({ limit: 10, offset: 0 });

    expect(pool.query).toHaveBeenCalledWith(
      expect.stringContaining('SELECT'),
      [10, 0]
    );
    expect(result).toEqual(mockData);
  });
});
```

### 6. Performance Monitoring

#### Response Time Logging
```javascript
// src/middleware/performance.js
const logger = require('../utils/logger');

const performanceMiddleware = (req, res, next) => {
  const startTime = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    logger.info('Request completed', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      userAgent: req.get('User-Agent')
    });
  });
  
  next();
};

module.exports = performanceMiddleware;
```

#### Database Query Performance
```javascript
// src/utils/queryProfiler.js
class QueryProfiler {
  static async profileQuery(queryName, queryFn) {
    const startTime = process.hrtime();
    
    try {
      const result = await queryFn();
      const [seconds, nanoseconds] = process.hrtime(startTime);
      const duration = seconds * 1000 + nanoseconds / 1000000;
      
      logger.info('Query performance', {
        queryName,
        duration: `${duration.toFixed(2)}ms`,
        rows: result.rows?.length || 0
      });
      
      return result;
    } catch (error) {
      const [seconds, nanoseconds] = process.hrtime(startTime);
      const duration = seconds * 1000 + nanoseconds / 1000000;
      
      logger.error('Query failed', {
        queryName,
        duration: `${duration.toFixed(2)}ms`,
        error: error.message
      });
      
      throw error;
    }
  }
}

module.exports = QueryProfiler;
```

### 7. Deployment Preparation

#### Production Configuration
```javascript
// src/config/production.js
module.exports = {
  port: process.env.PORT || 3000,
  database: {
    connectionString: process.env.DATABASE_URL,
    ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
    max: 20,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000
  },
  logging: {
    level: 'warn',
    format: 'json'
  },
  cors: {
    origin: process.env.CORS_ORIGIN?.split(',') || false,
    credentials: false
  }
};
```

#### Health Checks
```javascript
// src/routes/health.js
const express = require('express');
const router = express.Router();
const pool = require('../config/database');

router.get('/health', async (req, res) => {
  try {
    // Check database connection
    const dbCheck = await pool.query('SELECT NOW()');
    
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      checks: {
        database: 'connected',
        uptime: process.uptime()
      }
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      checks: {
        database: 'disconnected',
        error: error.message
      }
    });
  }
});

module.exports = router;
```

## Development Best Practices

1. **Use TypeScript** for better type safety (optional)
2. **Write tests first** (TDD approach)
3. **Use ESLint** for code quality
4. **Follow REST conventions** for endpoint design
5. **Implement proper logging** for debugging
6. **Use environment variables** for configuration
7. **Handle errors gracefully** with proper HTTP status codes
8. **Optimize database queries** with indexes and EXPLAIN ANALYZE
9. **Implement caching** for frequently accessed data
10. **Monitor performance** with appropriate metrics
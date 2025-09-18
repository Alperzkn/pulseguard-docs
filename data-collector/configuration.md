# Data Collector Configuration

## Environment Variables

The PulseGuard Data Collector uses environment variables for configuration. All settings are defined in the `.env` file.

### Database Configuration

```bash
# PostgreSQL connection string
DATABASE_URL=postgresql://username:password@host:5432/dbname

# Connection pool settings (optional)
DB_POOL_MAX=20
DB_POOL_IDLE_TIMEOUT=30000
DB_POOL_CONNECTION_TIMEOUT=2000
```

### CoinGecko API Configuration

```bash
# API Key (required)
COINGECKO_API_KEY=your_demo_api_key

# Base currency for price data
QUOTE=usd

# API timeout settings
API_TIMEOUT=10000
API_RETRIES=3
```

### Collection Settings

```bash
# Collection mode: top1000 | top5000 | complete
COLLECTION_MODE=top1000

# Number of coins per API request (max 250)
COINS_PER_BATCH=250

# Delay between batches (seconds)
BATCH_DELAY_SECONDS=6

# Priority coins (always included)
PRIORITY_COINS=bitcoin,ethereum,cardano
```

### Rate Limiting Configuration

```bash
# Maximum retries for failed requests
MAX_RETRIES=3

# Backoff multiplier for exponential backoff
BACKOFF_MULTIPLIER=2

# Base delay for backoff (milliseconds)
BASE_DELAY=1000

# Maximum delay for backoff (milliseconds)
MAX_DELAY=30000
```

### Priority Management

```bash
# Hour for daily priority refresh (0-23, UTC)
PRIORITY_REFRESH_HOUR=0

# Priority cache file location
PRIORITY_CACHE_FILE=data/coin_priorities.json

# Force priority refresh on startup
FORCE_PRIORITY_REFRESH=false
```

### Logging Configuration

```bash
# Log level: debug | info | warn | error
LOG_LEVEL=info

# Log to file
LOG_TO_FILE=true

# Log file path
LOG_FILE_PATH=logs/collector.log

# Maximum log file size (MB)
LOG_MAX_SIZE=10

# Maximum number of log files to keep
LOG_MAX_FILES=5
```

## Configuration Files

### Main Configuration (`app/config.js`)

```javascript
const config = {
  // Database settings
  database: {
    url: process.env.DATABASE_URL,
    pool: {
      max: parseInt(process.env.DB_POOL_MAX) || 20,
      idleTimeoutMillis: parseInt(process.env.DB_POOL_IDLE_TIMEOUT) || 30000,
      connectionTimeoutMillis: parseInt(process.env.DB_POOL_CONNECTION_TIMEOUT) || 2000
    }
  },

  // CoinGecko API settings
  coingecko: {
    apiKey: process.env.COINGECKO_API_KEY,
    baseUrl: 'https://api.coingecko.com/api/v3',
    quote: process.env.QUOTE || 'usd',
    timeout: parseInt(process.env.API_TIMEOUT) || 10000,
    retries: parseInt(process.env.API_RETRIES) || 3
  },

  // Collection settings
  collection: {
    mode: process.env.COLLECTION_MODE || 'top1000',
    coinsPerBatch: parseInt(process.env.COINS_PER_BATCH) || 250,
    batchDelay: parseInt(process.env.BATCH_DELAY_SECONDS) || 6,
    priorityCoins: (process.env.PRIORITY_COINS || 'bitcoin,ethereum,cardano').split(',')
  },

  // Rate limiting
  rateLimiting: {
    maxRetries: parseInt(process.env.MAX_RETRIES) || 3,
    backoffMultiplier: parseFloat(process.env.BACKOFF_MULTIPLIER) || 2,
    baseDelay: parseInt(process.env.BASE_DELAY) || 1000,
    maxDelay: parseInt(process.env.MAX_DELAY) || 30000
  },

  // Priority management
  priority: {
    refreshHour: parseInt(process.env.PRIORITY_REFRESH_HOUR) || 0,
    cacheFile: process.env.PRIORITY_CACHE_FILE || 'data/coin_priorities.json',
    forceRefresh: process.env.FORCE_PRIORITY_REFRESH === 'true'
  },

  // Logging
  logging: {
    level: process.env.LOG_LEVEL || 'info',
    toFile: process.env.LOG_TO_FILE === 'true',
    filePath: process.env.LOG_FILE_PATH || 'logs/collector.log',
    maxSize: (parseInt(process.env.LOG_MAX_SIZE) || 10) * 1024 * 1024,
    maxFiles: parseInt(process.env.LOG_MAX_FILES) || 5
  }
};

module.exports = config;
```

## Collection Mode Configurations

### Top 1000 Mode (Default)

```bash
COLLECTION_MODE=top1000
COINS_PER_BATCH=250
BATCH_DELAY_SECONDS=6
```

**Characteristics:**
- Target: Top 1000 cryptocurrencies by market cap
- API Calls: 4 requests per cycle (1000 ÷ 250)
- Collection Time: ~4-16 minutes
- Coverage: ~99% of total market capitalization

### Top 5000 Mode

```bash
COLLECTION_MODE=top5000
COINS_PER_BATCH=250
BATCH_DELAY_SECONDS=6
```

**Characteristics:**
- Target: Top 5000 cryptocurrencies by market cap
- API Calls: 20 requests per cycle (5000 ÷ 250)
- Collection Time: ~20-80 minutes
- Coverage: ~99.9% of total market capitalization

### Complete Mode

```bash
COLLECTION_MODE=complete
COINS_PER_BATCH=250
BATCH_DELAY_SECONDS=6
```

**Characteristics:**
- Target: All available cryptocurrencies (~18,000+)
- API Calls: 75+ requests per cycle
- Collection Time: ~75-300 minutes
- Coverage: 100% of available data

## Environment-Specific Configurations

### Development Environment

```bash
# .env.development
DATABASE_URL=postgresql://pulseguard_user:password@localhost:5432/crypto_db_dev
COINGECKO_API_KEY=your_demo_api_key
COLLECTION_MODE=top1000
BATCH_DELAY_SECONDS=10
LOG_LEVEL=debug
LOG_TO_FILE=false
FORCE_PRIORITY_REFRESH=true
```

### Staging Environment

```bash
# .env.staging
DATABASE_URL=postgresql://pulseguard_user:password@staging-db:5432/crypto_db_staging
COINGECKO_API_KEY=your_demo_api_key
COLLECTION_MODE=top5000
BATCH_DELAY_SECONDS=6
LOG_LEVEL=info
LOG_TO_FILE=true
PRIORITY_REFRESH_HOUR=1
```

### Production Environment

```bash
# .env.production
DATABASE_URL=postgresql://pulseguard_user:password@prod-db:5432/crypto_db
COINGECKO_API_KEY=your_production_api_key
COLLECTION_MODE=top5000
BATCH_DELAY_SECONDS=2.5
LOG_LEVEL=warn
LOG_TO_FILE=true
LOG_FILE_PATH=/var/log/pulseguard/collector.log
PRIORITY_REFRESH_HOUR=0
MAX_RETRIES=5
```

## CoinGecko API Plan Configurations

### Demo Plan Configuration

```bash
# Demo Plan: 30 calls/minute, 10K calls/month
COINGECKO_API_KEY=your_demo_key
BATCH_DELAY_SECONDS=6
MAX_RETRIES=3
COLLECTION_MODE=top1000
```

### Pro Plan Configuration

```bash
# Pro Plan: 500 calls/minute, 100K calls/month
COINGECKO_API_KEY=your_pro_key
BATCH_DELAY_SECONDS=2
MAX_RETRIES=5
COLLECTION_MODE=top5000
```

### Enterprise Plan Configuration

```bash
# Enterprise Plan: 1000+ calls/minute
COINGECKO_API_KEY=your_enterprise_key
BATCH_DELAY_SECONDS=1
MAX_RETRIES=3
COLLECTION_MODE=complete
```

## Configuration Validation

### Environment Validation Script

```javascript
// scripts/validate-config.js
const config = require('../app/config');

function validateConfig() {
  const errors = [];

  // Required environment variables
  if (!config.database.url) {
    errors.push('DATABASE_URL is required');
  }

  if (!config.coingecko.apiKey) {
    errors.push('COINGECKO_API_KEY is required');
  }

  // Validate collection mode
  const validModes = ['top1000', 'top5000', 'complete'];
  if (!validModes.includes(config.collection.mode)) {
    errors.push(`COLLECTION_MODE must be one of: ${validModes.join(', ')}`);
  }

  // Validate batch size
  if (config.collection.coinsPerBatch > 250) {
    errors.push('COINS_PER_BATCH cannot exceed 250 (CoinGecko API limit)');
  }

  // Validate priority refresh hour
  if (config.priority.refreshHour < 0 || config.priority.refreshHour > 23) {
    errors.push('PRIORITY_REFRESH_HOUR must be between 0 and 23');
  }

  // Validate log level
  const validLogLevels = ['debug', 'info', 'warn', 'error'];
  if (!validLogLevels.includes(config.logging.level)) {
    errors.push(`LOG_LEVEL must be one of: ${validLogLevels.join(', ')}`);
  }

  return errors;
}

const errors = validateConfig();
if (errors.length > 0) {
  console.error('Configuration validation failed:');
  errors.forEach(error => console.error(`  - ${error}`));
  process.exit(1);
} else {
  console.log('✅ Configuration is valid');
}
```

## Dynamic Configuration

### Runtime Configuration Updates

```javascript
// app/dynamic-config.js
class DynamicConfig {
  constructor(baseConfig) {
    this.config = { ...baseConfig };
    this.listeners = new Map();
  }

  // Update batch delay based on API response times
  adjustBatchDelay(responseTime) {
    if (responseTime > 5000) {
      // Slow responses, increase delay
      this.config.collection.batchDelay = Math.min(
        this.config.collection.batchDelay * 1.5,
        30
      );
    } else if (responseTime < 1000) {
      // Fast responses, decrease delay
      this.config.collection.batchDelay = Math.max(
        this.config.collection.batchDelay * 0.9,
        1
      );
    }
  }

  // Adjust retries based on error rate
  adjustRetries(errorRate) {
    if (errorRate > 0.1) {
      // High error rate, increase retries
      this.config.rateLimiting.maxRetries = Math.min(
        this.config.rateLimiting.maxRetries + 1,
        10
      );
    } else if (errorRate < 0.01) {
      // Low error rate, decrease retries
      this.config.rateLimiting.maxRetries = Math.max(
        this.config.rateLimiting.maxRetries - 1,
        1
      );
    }
  }

  // Get current configuration
  get() {
    return { ...this.config };
  }

  // Subscribe to configuration changes
  onChange(key, callback) {
    if (!this.listeners.has(key)) {
      this.listeners.set(key, []);
    }
    this.listeners.get(key).push(callback);
  }

  // Notify listeners of changes
  notifyChange(key, oldValue, newValue) {
    const callbacks = this.listeners.get(key) || [];
    callbacks.forEach(callback => callback(oldValue, newValue));
  }
}

module.exports = DynamicConfig;
```

## Configuration Monitoring

### Configuration Health Check

```javascript
// app/config-monitor.js
class ConfigMonitor {
  constructor(config) {
    this.config = config;
    this.checks = new Map();
  }

  // Monitor API key validity
  async checkApiKey() {
    try {
      const response = await fetch(
        `${this.config.coingecko.baseUrl}/ping`,
        {
          headers: {
            'X-Cg-Demo-Api-Key': this.config.coingecko.apiKey
          }
        }
      );
      return response.ok;
    } catch (error) {
      return false;
    }
  }

  // Monitor database connectivity
  async checkDatabase() {
    try {
      const pool = require('./database');
      const result = await pool.query('SELECT NOW()');
      return result.rows.length > 0;
    } catch (error) {
      return false;
    }
  }

  // Monitor disk space for logs
  async checkDiskSpace() {
    if (!this.config.logging.toFile) return true;
    
    try {
      const fs = require('fs');
      const stats = fs.statSync(this.config.logging.filePath);
      const diskSpace = stats.size;
      return diskSpace < this.config.logging.maxSize;
    } catch (error) {
      return true; // File doesn't exist yet
    }
  }

  // Run all health checks
  async healthCheck() {
    const results = {
      apiKey: await this.checkApiKey(),
      database: await this.checkDatabase(),
      diskSpace: await this.checkDiskSpace(),
      timestamp: new Date().toISOString()
    };

    return {
      healthy: Object.values(results).every(check => check),
      checks: results
    };
  }
}

module.exports = ConfigMonitor;
```

## Best Practices

1. **Environment Separation**: Use different configurations for development, staging, and production
2. **Secret Management**: Never commit API keys or passwords to version control
3. **Validation**: Always validate configuration on startup
4. **Monitoring**: Monitor configuration health and API limits
5. **Documentation**: Keep configuration documentation up to date
6. **Defaults**: Provide sensible defaults for optional settings
7. **Type Safety**: Validate data types for numeric configurations
8. **Dynamic Adjustment**: Allow runtime adjustment of non-critical settings
9. **Backup**: Keep backup configurations for rollback scenarios
10. **Logging**: Log configuration changes for audit trails
# API Reference

This document describes the internal API structure and database schema for the PulseGuard cryptocurrency data collector.

## Database Schema

### Core Data Tables

#### `minute_close`
Stores minute-by-minute price data with 60-minute retention.

```sql
CREATE TABLE minute_close (
    coin_id TEXT NOT NULL,
    minute_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, minute_ts)
);
```

#### `hourly_close` 
Stores hourly aggregated data with 24-hour retention.

```sql
CREATE TABLE hourly_close (
    coin_id TEXT NOT NULL,
    hour_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, hour_ts)
);
```

#### `daily_close`
Stores daily aggregated data with 7-day retention.

```sql
CREATE TABLE daily_close (
    coin_id TEXT NOT NULL,
    day_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, day_ts)
);
```

#### `weekly_close`
Stores weekly aggregated data with 52-week retention.

```sql
CREATE TABLE weekly_close (
    coin_id TEXT NOT NULL,
    week_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, week_ts)
);
```

#### `monthly_close`
Stores monthly aggregated data with permanent retention.

```sql
CREATE TABLE monthly_close (
    coin_id TEXT NOT NULL,
    month_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, month_ts)
);
```

### Management Tables

#### `coin_priorities`
Tracks market cap rankings and priority management.

```sql
CREATE TABLE coin_priorities (
    coin_id TEXT PRIMARY KEY,
    market_cap_rank INTEGER NOT NULL,
    market_cap_usd NUMERIC(20,2),
    last_updated TIMESTAMPTZ DEFAULT NOW()
);
```

#### `collection_status`
Monitors collection health and metrics.

```sql
CREATE TABLE collection_status (
    id SERIAL PRIMARY KEY,
    collection_type TEXT NOT NULL,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ,
    coins_processed INTEGER DEFAULT 0,
    coins_failed INTEGER DEFAULT 0,
    status TEXT DEFAULT 'running',
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Core Modules

### Priority Manager (`app/priority-manager.js`)

Manages market cap rankings and coin prioritization.

**Key Functions:**
- `refreshPriorities()` - Fetches latest market cap rankings from CoinGecko
- `getCoinBatches(mode)` - Generates batched coin lists based on collection mode
- `updateCoinRanks()` - Updates database with latest rankings

**Configuration:**
```javascript
const modes = {
    top1000: { limit: 1000, batches: 4 },
    top5000: { limit: 5000, batches: 20 },
    complete: { limit: null, batches: 75 }
};
```

### Complete Enhanced Collector (`app/complete-enhanced-collector.js`)

Main collection engine for batch processing with rate limiting.

**Key Functions:**
- `collectPrices()` - Main collection orchestrator
- `processBatch(coins)` - Processes single batch of coins
- `aggregateTimeframes()` - Creates hourly/daily/weekly/monthly aggregations
- `cleanupOldData()` - Removes expired data based on retention policies

**Rate Limiting:**
- Configurable batch delays (default: 6 seconds)
- Exponential backoff for 429 errors
- Maximum 3 retries per batch

### Continuous Collector (`app/continuous-collector.js`)

Handles continuous minute-by-minute collection with queue management.

**Key Features:**
- Missed minute recovery
- Automatic queue processing
- Gap detection and filling
- Health monitoring

## Configuration

### Environment Variables

```bash
# Collection Settings
COLLECTION_MODE=top1000              # top1000 | top5000 | complete
COINS_PER_BATCH=250                  # Coins per API request
BATCH_DELAY_SECONDS=6                # Delay between batches

# CoinGecko API
COINGECKO_API_KEY=your_demo_api_key
QUOTE=usd

# Database
DATABASE_URL=postgresql://user:pass@host:5432/dbname

# Priority Management
PRIORITY_COINS=bitcoin,ethereum,cardano
PRIORITY_REFRESH_HOUR=0
```

### Collection Modes

| Mode | Coins | Batches | Time Estimate | Coverage |
|------|-------|---------|---------------|----------|
| top1000 | 1,000 | 4 | 4-16 min | ~99% market cap |
| top5000 | 5,000 | 20 | 20-80 min | ~99.9% market cap |
| complete | 18,575+ | 75+ | 75-300 min | 100% coverage |

## API Endpoints (CoinGecko)

### Primary Data Source
```
GET /api/v3/simple/price
Parameters:
  - ids: comma-separated coin IDs (max 250)
  - vs_currencies: usd
  - include_market_cap: true
  - include_24hr_change: true
```

### Market Rankings
```
GET /api/v3/coins/markets
Parameters:
  - vs_currency: usd
  - order: market_cap_desc
  - per_page: 250
  - page: 1,2,3...
  - sparkline: false
```

## Data Flow

1. **Priority Refresh** (Daily at 00:00 UTC)
   - Fetch latest market cap rankings
   - Update `coin_priorities` table
   - Generate new batch configurations

2. **Collection Cycle** (Every minute or on-demand)
   - Load coin batches based on mode
   - Process each batch with rate limiting
   - Store raw data in `minute_close`
   - Trigger aggregations if needed

3. **Aggregation Process**
   - Hourly: Last minute of each hour → `hourly_close`
   - Daily: Last minute of each day → `daily_close`
   - Weekly: Last minute of ISO week → `weekly_close`
   - Monthly: Last minute of each month → `monthly_close`

4. **Cleanup Process**
   - Remove `minute_close` data > 60 minutes
   - Remove `hourly_close` data > 24 hours
   - Remove `daily_close` data > 7 days
   - Remove `weekly_close` data > 52 weeks
   - Keep `monthly_close` data permanently

## Performance Indexes

```sql
-- Primary performance indexes
CREATE INDEX idx_coin_priorities_rank ON coin_priorities (market_cap_rank);
CREATE INDEX idx_minute_close_rank ON minute_close (coin_rank, minute_ts DESC);
CREATE INDEX idx_minute_close_ts ON minute_close (minute_ts DESC);
CREATE INDEX idx_hourly_close_ts ON hourly_close (hour_ts DESC);
CREATE INDEX idx_daily_close_ts ON daily_close (day_ts DESC);
CREATE INDEX idx_weekly_close_ts ON weekly_close (week_ts DESC);
CREATE INDEX idx_monthly_close_ts ON monthly_close (month_ts DESC);
```

## Monitoring

### Key Metrics
- Collection completion time
- API error rates by batch
- Successful coin updates per cycle
- Priority refresh status
- Database storage usage

### Health Checks
- Last successful collection timestamp
- API connectivity status
- Database connection health
- Queue depth for continuous mode

---

*Last updated: 2025-09-18*
*Version: 1.1*
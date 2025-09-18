# Cryptocurrency Collection Strategy

## Overview
This document outlines the systematic approach for collecting cryptocurrency price data from CoinGecko API, maximizing coverage while respecting API rate limits.

## Current State
- **Total available cryptocurrencies:** 18,575 coins in CoinGecko
- **Current collection:** 3 coins (bitcoin, ethereum, cardano)
- **API Rate Limits:** 
  - Free tier: 5-15 calls/min (unstable)
  - Demo tier: 30 calls/min, 10K calls/month (recommended)
  - Current batch size: 250 coins per request

## Collection Strategy

### 1. Market Cap Priority System
- **Primary sorting:** Market capitalization ranking (highest to lowest)
- **Priority update frequency:** Daily refresh at 00:00 UTC
- **Fallback coins:** Always include bitcoin, ethereum, cardano regardless of ranking
- **Ranking source:** CoinGecko `/coins/markets` endpoint with `order=market_cap_desc`

### 2. Phased Collection Approach

#### Phase 1: Top 1000 Cryptocurrencies (Default)
```
Batches: 4 API calls (1000 coins ÷ 250 per batch)
Coverage: ~99% of total market capitalization
Collection time: 4-16 minutes (depending on rate limits)
```

#### Phase 2: Extended Coverage - Top 5000
```
Batches: 20 API calls (5000 coins ÷ 250 per batch)  
Coverage: ~99.9% of total market capitalization
Collection time: 20-80 minutes
```

#### Phase 3: Complete Coverage - All Coins
```
Batches: 75 API calls (18,575 coins ÷ 250 per batch)
Coverage: 100% of available cryptocurrencies
Collection time: 75-300 minutes
```

### 3. Configuration Management

#### Environment Variables
```bash
# Collection scope
COLLECTION_MODE=top1000          # "top1000" | "top5000" | "complete"
COINS_PER_BATCH=250             # Maximum coins per API request
PRIORITY_COINS=bitcoin,ethereum,cardano  # Always included coins

# Rate limiting
BATCH_DELAY_SECONDS=4           # Delay between batches (15 calls/min)
MAX_RETRIES=3                   # Retry failed requests
BACKOFF_MULTIPLIER=2            # Exponential backoff for 429 errors

# Priority updates  
PRIORITY_REFRESH_HOUR=0         # Daily refresh at midnight UTC
PRIORITY_CACHE_FILE=data/coin_priorities.json
```

#### Collection Modes
- **top1000:** Collect top 1000 coins by market cap (recommended)
- **top5000:** Extended coverage for comprehensive analysis
- **complete:** Full coverage of all available cryptocurrencies

### 4. Daily Priority Refresh System

#### Process Flow
1. **Daily trigger:** At 00:00 UTC, fetch latest market data
2. **Ranking update:** Get coins sorted by market cap from `/coins/markets`
3. **Priority cache:** Save ranked coin list to `data/coin_priorities.json`
4. **Batch generation:** Create paginated batches based on COLLECTION_MODE
5. **Fallback handling:** Ensure priority coins are always in first batch

#### Market Data Source
```
Endpoint: GET /coins/markets
Parameters:
  - vs_currency=usd
  - order=market_cap_desc  
  - per_page=250
  - page=1,2,3... (paginated)
  - sparkline=false
```

### 5. Rate Limiting Strategy

#### Conservative Approach (Free Tier)
```
Rate limit: 10 calls/minute maximum
Batch delay: 6 seconds between requests
Top 1000 coins: ~4 minutes per collection cycle
```

#### Optimized Approach (Demo Tier)  
```
Rate limit: 25 calls/minute
Batch delay: 2.5 seconds between requests  
Top 1000 coins: ~1 minute per collection cycle
```

### 6. Error Handling & Resilience

#### Individual Coin Failures
- Continue processing remaining coins in batch
- Log failed coins for later retry
- Don't stop entire collection for single coin issues

#### API Rate Limit Handling
- Detect 429 responses
- Implement exponential backoff (2x, 4x, 8x delays)
- Maximum 3 retries per batch

#### Network/Service Failures  
- Graceful degradation to priority coins only
- Resume from last successful batch
- Health monitoring endpoint

### 7. Database Schema Enhancements

#### New Fields
```sql
-- Add to existing tables
ALTER TABLE minute_close ADD COLUMN coin_rank INTEGER;
ALTER TABLE hourly_close ADD COLUMN coin_rank INTEGER;
-- ... (similar for other tables)

-- Priority tracking table
CREATE TABLE coin_priorities (
  coin_id TEXT PRIMARY KEY,
  market_cap_rank INTEGER NOT NULL,
  market_cap_usd NUMERIC(20,2),
  last_updated TIMESTAMPTZ DEFAULT NOW()
);
```

#### Indexing Strategy
```sql
-- Performance optimization for ranked queries
CREATE INDEX idx_coin_priorities_rank ON coin_priorities (market_cap_rank);
CREATE INDEX idx_minute_close_rank ON minute_close (coin_rank, minute_ts DESC);
```

### 8. Implementation Components

#### A. Priority Manager (`app/priority-manager.js`)
- Fetch and cache daily market cap rankings
- Generate paginated coin batches  
- Handle priority coin inclusion
- Manage ranking updates

#### B. Enhanced Collector (`app/enhanced-collector.js`)
- Process multiple batches with rate limiting
- Handle batch-level error recovery
- Progress tracking and resumption
- Detailed logging and metrics

#### C. Configuration Loader (`app/config.js`)
- Load and validate environment settings
- Provide collection mode configurations
- Handle priority cache loading

### 9. Monitoring & Observability

#### Key Metrics
- Collection completion time
- API error rates by batch
- Number of successful coin updates
- Priority refresh success/failure

#### Logging Strategy
- INFO: Collection start/completion, batch progress
- WARN: Individual coin failures, rate limit hits
- ERROR: Complete collection failures, priority refresh issues

### 10. Deployment Considerations

#### Kubernetes CronJob Updates
- Adjust resource limits based on collection mode
- Set appropriate timeout values (up to 300 minutes for complete mode)
- Configure persistent volume for priority cache

#### Environment-Specific Settings
```yaml
# Development
COLLECTION_MODE=top1000
BATCH_DELAY_SECONDS=6

# Production  
COLLECTION_MODE=top5000
BATCH_DELAY_SECONDS=2.5
```

## Implementation Timeline

1. **Week 1:** Priority manager and database schema updates
2. **Week 2:** Enhanced collector with batch processing
3. **Week 3:** Daily priority refresh automation
4. **Week 4:** Testing, monitoring, and production deployment

## Risk Mitigation

- **API limit exceeded:** Automatic fallback to smaller coin sets
- **Ranking service failure:** Use cached priorities from previous day
- **Database performance:** Implement coin_rank indexing and partitioning
- **Memory usage:** Stream processing for large coin lists

---

*Last updated: 2025-09-04*
*Version: 1.0*
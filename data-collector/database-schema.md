# Database Schema

## Overview

The PulseGuard database schema is designed for efficient time-series data storage with multiple aggregation levels. It uses PostgreSQL with time-based partitioning for optimal performance.

## Schema Design Principles

1. **Time-Series Optimization**: Data is stored in time-based tables with automatic partitioning
2. **Multi-Timeframe Support**: Separate tables for different aggregation levels
3. **Performance Indexing**: Composite indexes for fast querying
4. **Data Retention**: Automatic cleanup based on retention policies
5. **Scalability**: Designed to handle millions of records efficiently

## Core Tables

### Time-Series Data Tables

#### `minute_close`
Stores minute-by-minute price data with 60-minute retention.

```sql
CREATE TABLE minute_close (
    coin_id TEXT NOT NULL,
    minute_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8) NOT NULL,
    market_cap_usd NUMERIC(20,2),
    volume_24h_usd NUMERIC(20,2),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, minute_ts)
) PARTITION BY RANGE (minute_ts);

-- Create monthly partitions
CREATE TABLE minute_close_y2025m01 PARTITION OF minute_close
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE minute_close_y2025m02 PARTITION OF minute_close
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

#### `hourly_close`
Stores hourly aggregated data with 24-hour retention.

```sql
CREATE TABLE hourly_close (
    coin_id TEXT NOT NULL,
    hour_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8) NOT NULL,
    market_cap_usd NUMERIC(20,2),
    volume_24h_usd NUMERIC(20,2),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, hour_ts)
) PARTITION BY RANGE (hour_ts);
```

#### `daily_close`
Stores daily aggregated data with 7-day retention.

```sql
CREATE TABLE daily_close (
    coin_id TEXT NOT NULL,
    day_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8) NOT NULL,
    market_cap_usd NUMERIC(20,2),
    volume_24h_usd NUMERIC(20,2),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, day_ts)
) PARTITION BY RANGE (day_ts);
```

#### `weekly_close`
Stores weekly aggregated data with 52-week retention.

```sql
CREATE TABLE weekly_close (
    coin_id TEXT NOT NULL,
    week_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8) NOT NULL,
    market_cap_usd NUMERIC(20,2),
    volume_24h_usd NUMERIC(20,2),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, week_ts)
) PARTITION BY RANGE (week_ts);
```

#### `monthly_close`
Stores monthly aggregated data with permanent retention.

```sql
CREATE TABLE monthly_close (
    coin_id TEXT NOT NULL,
    month_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8) NOT NULL,
    market_cap_usd NUMERIC(20,2),
    volume_24h_usd NUMERIC(20,2),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, month_ts)
) PARTITION BY RANGE (month_ts);
```

### Management Tables

#### `coin_priorities`
Tracks market cap rankings and coin metadata.

```sql
CREATE TABLE coin_priorities (
    coin_id TEXT PRIMARY KEY,
    market_cap_rank INTEGER NOT NULL,
    market_cap_usd NUMERIC(20,2),
    symbol TEXT,
    name TEXT,
    image_url TEXT,
    last_updated TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_coin_priorities_rank ON coin_priorities (market_cap_rank);
CREATE INDEX idx_coin_priorities_updated ON coin_priorities (last_updated DESC);
```

#### `collection_status`
Monitors collection health and metrics.

```sql
CREATE TABLE collection_status (
    id SERIAL PRIMARY KEY,
    collection_type TEXT NOT NULL, -- 'complete', 'continuous', 'priority_refresh'
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ,
    coins_processed INTEGER DEFAULT 0,
    coins_failed INTEGER DEFAULT 0,
    api_calls_made INTEGER DEFAULT 0,
    status TEXT DEFAULT 'running', -- 'running', 'completed', 'failed', 'cancelled'
    error_message TEXT,
    batch_details JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for monitoring queries
CREATE INDEX idx_collection_status_start_time ON collection_status (start_time DESC);
CREATE INDEX idx_collection_status_type_status ON collection_status (collection_type, status);
```

#### `api_usage_stats`
Tracks CoinGecko API usage for rate limit monitoring.

```sql
CREATE TABLE api_usage_stats (
    id SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    total_calls INTEGER DEFAULT 0,
    successful_calls INTEGER DEFAULT 0,
    failed_calls INTEGER DEFAULT 0,
    rate_limited_calls INTEGER DEFAULT 0,
    average_response_time INTEGER, -- in milliseconds
    peak_calls_per_minute INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(date)
);

-- Index for daily usage queries
CREATE INDEX idx_api_usage_date ON api_usage_stats (date DESC);
```

## Performance Indexes

### Time-Series Query Optimization

```sql
-- Composite indexes for efficient time-series queries
CREATE INDEX CONCURRENTLY idx_minute_close_coin_time 
    ON minute_close (coin_id, minute_ts DESC);

CREATE INDEX CONCURRENTLY idx_hourly_close_coin_time 
    ON hourly_close (coin_id, hour_ts DESC);

CREATE INDEX CONCURRENTLY idx_daily_close_coin_time 
    ON daily_close (coin_id, day_ts DESC);

CREATE INDEX CONCURRENTLY idx_weekly_close_coin_time 
    ON weekly_close (coin_id, week_ts DESC);

CREATE INDEX CONCURRENTLY idx_monthly_close_coin_time 
    ON monthly_close (coin_id, month_ts DESC);

-- Time-only indexes for aggregation queries
CREATE INDEX CONCURRENTLY idx_minute_close_time 
    ON minute_close (minute_ts DESC);

CREATE INDEX CONCURRENTLY idx_hourly_close_time 
    ON hourly_close (hour_ts DESC);

CREATE INDEX CONCURRENTLY idx_daily_close_time 
    ON daily_close (day_ts DESC);

CREATE INDEX CONCURRENTLY idx_weekly_close_time 
    ON weekly_close (week_ts DESC);

CREATE INDEX CONCURRENTLY idx_monthly_close_time 
    ON monthly_close (month_ts DESC);
```

### Partial Indexes for Recent Data

```sql
-- Partial indexes for frequently accessed recent data
CREATE INDEX CONCURRENTLY idx_recent_minute_close 
    ON minute_close (minute_ts DESC, coin_id)
    WHERE minute_ts > NOW() - INTERVAL '2 hours';

CREATE INDEX CONCURRENTLY idx_recent_hourly_close 
    ON hourly_close (hour_ts DESC, coin_id)
    WHERE hour_ts > NOW() - INTERVAL '48 hours';

-- Index for market cap rankings
CREATE INDEX CONCURRENTLY idx_coin_priorities_rank_active 
    ON coin_priorities (market_cap_rank)
    WHERE market_cap_rank <= 1000;
```

## Data Relationships

### Foreign Key Constraints

```sql
-- Ensure data integrity between tables
ALTER TABLE minute_close 
    ADD CONSTRAINT fk_minute_close_coin 
    FOREIGN KEY (coin_id) REFERENCES coin_priorities(coin_id);

ALTER TABLE hourly_close 
    ADD CONSTRAINT fk_hourly_close_coin 
    FOREIGN KEY (coin_id) REFERENCES coin_priorities(coin_id);

ALTER TABLE daily_close 
    ADD CONSTRAINT fk_daily_close_coin 
    FOREIGN KEY (coin_id) REFERENCES coin_priorities(coin_id);

ALTER TABLE weekly_close 
    ADD CONSTRAINT fk_weekly_close_coin 
    FOREIGN KEY (coin_id) REFERENCES coin_priorities(coin_id);

ALTER TABLE monthly_close 
    ADD CONSTRAINT fk_monthly_close_coin 
    FOREIGN KEY (coin_id) REFERENCES coin_priorities(coin_id);
```

### Check Constraints

```sql
-- Data validation constraints
ALTER TABLE coin_priorities 
    ADD CONSTRAINT chk_market_cap_rank_positive 
    CHECK (market_cap_rank > 0);

ALTER TABLE coin_priorities 
    ADD CONSTRAINT chk_market_cap_positive 
    CHECK (market_cap_usd >= 0);

ALTER TABLE minute_close 
    ADD CONSTRAINT chk_price_positive 
    CHECK (price_usd > 0);

ALTER TABLE collection_status 
    ADD CONSTRAINT chk_valid_status 
    CHECK (status IN ('running', 'completed', 'failed', 'cancelled'));

ALTER TABLE collection_status 
    ADD CONSTRAINT chk_valid_collection_type 
    CHECK (collection_type IN ('complete', 'continuous', 'priority_refresh', 'manual'));
```

## Data Partitioning Strategy

### Automatic Partition Creation

```sql
-- Function to create monthly partitions automatically
CREATE OR REPLACE FUNCTION create_monthly_partition(
    table_name TEXT, 
    start_date DATE
) RETURNS VOID AS $$
DECLARE
    partition_name TEXT;
    end_date DATE;
BEGIN
    end_date := start_date + INTERVAL '1 month';
    partition_name := table_name || '_y' || EXTRACT(YEAR FROM start_date) || 'm' || 
                     LPAD(EXTRACT(MONTH FROM start_date)::TEXT, 2, '0');
    
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I 
         FOR VALUES FROM (%L) TO (%L)',
        partition_name, table_name, start_date, end_date
    );
END;
$$ LANGUAGE plpgsql;

-- Create partitions for the next 12 months
DO $$
DECLARE
    current_date DATE := DATE_TRUNC('month', CURRENT_DATE);
    i INTEGER;
BEGIN
    FOR i IN 0..11 LOOP
        PERFORM create_monthly_partition('minute_close', current_date + (i || ' months')::INTERVAL);
        PERFORM create_monthly_partition('hourly_close', current_date + (i || ' months')::INTERVAL);
        PERFORM create_monthly_partition('daily_close', current_date + (i || ' months')::INTERVAL);
        PERFORM create_monthly_partition('weekly_close', current_date + (i || ' months')::INTERVAL);
        PERFORM create_monthly_partition('monthly_close', current_date + (i || ' months')::INTERVAL);
    END LOOP;
END
$$;
```

### Partition Maintenance

```sql
-- Function to drop old partitions
CREATE OR REPLACE FUNCTION drop_old_partitions(
    table_name TEXT,
    retention_months INTEGER
) RETURNS INTEGER AS $$
DECLARE
    partition_record RECORD;
    dropped_count INTEGER := 0;
    cutoff_date DATE;
BEGIN
    cutoff_date := DATE_TRUNC('month', CURRENT_DATE) - (retention_months || ' months')::INTERVAL;
    
    FOR partition_record IN 
        SELECT schemaname, tablename 
        FROM pg_tables 
        WHERE tablename LIKE table_name || '_y%'
    LOOP
        -- Extract date from partition name and check if it's old enough
        DECLARE
            partition_date DATE;
        BEGIN
            partition_date := TO_DATE(
                SUBSTRING(partition_record.tablename FROM '.*_y(\d{4})m(\d{2})'),
                'YYYYMM'
            );
            
            IF partition_date < cutoff_date THEN
                EXECUTE format('DROP TABLE IF EXISTS %I.%I', 
                              partition_record.schemaname, 
                              partition_record.tablename);
                dropped_count := dropped_count + 1;
            END IF;
        END;
    END LOOP;
    
    RETURN dropped_count;
END;
$$ LANGUAGE plpgsql;
```

## Data Aggregation Triggers

### Automatic Hourly Aggregation

```sql
-- Function to aggregate minute data to hourly
CREATE OR REPLACE FUNCTION aggregate_to_hourly()
RETURNS TRIGGER AS $$
DECLARE
    hour_timestamp TIMESTAMPTZ;
    coin_data RECORD;
BEGIN
    -- Calculate the hour timestamp
    hour_timestamp := DATE_TRUNC('hour', NEW.minute_ts);
    
    -- Get the latest data for this coin and hour
    SELECT 
        NEW.coin_id as coin_id,
        hour_timestamp as hour_ts,
        price_usd,
        market_cap_usd,
        volume_24h_usd,
        coin_rank
    INTO coin_data
    FROM minute_close
    WHERE coin_id = NEW.coin_id 
      AND minute_ts >= hour_timestamp
      AND minute_ts < hour_timestamp + INTERVAL '1 hour'
    ORDER BY minute_ts DESC
    LIMIT 1;
    
    -- Insert or update hourly data
    INSERT INTO hourly_close (
        coin_id, hour_ts, price_usd, market_cap_usd, volume_24h_usd, coin_rank
    ) VALUES (
        coin_data.coin_id, coin_data.hour_ts, coin_data.price_usd,
        coin_data.market_cap_usd, coin_data.volume_24h_usd, coin_data.coin_rank
    )
    ON CONFLICT (coin_id, hour_ts) 
    DO UPDATE SET
        price_usd = EXCLUDED.price_usd,
        market_cap_usd = EXCLUDED.market_cap_usd,
        volume_24h_usd = EXCLUDED.volume_24h_usd,
        coin_rank = EXCLUDED.coin_rank,
        created_at = NOW();
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger for automatic aggregation
CREATE TRIGGER trigger_aggregate_hourly
    AFTER INSERT ON minute_close
    FOR EACH ROW
    EXECUTE FUNCTION aggregate_to_hourly();
```

## Data Cleanup Functions

### Retention Policy Enforcement

```sql
-- Function to clean up old data based on retention policies
CREATE OR REPLACE FUNCTION cleanup_old_data()
RETURNS TABLE(table_name TEXT, rows_deleted BIGINT) AS $$
DECLARE
    result_record RECORD;
BEGIN
    -- Clean minute_close (60 minutes retention)
    DELETE FROM minute_close 
    WHERE minute_ts < NOW() - INTERVAL '60 minutes';
    GET DIAGNOSTICS result_record.rows_deleted = ROW_COUNT;
    
    table_name := 'minute_close';
    rows_deleted := result_record.rows_deleted;
    RETURN NEXT;
    
    -- Clean hourly_close (24 hours retention)
    DELETE FROM hourly_close 
    WHERE hour_ts < NOW() - INTERVAL '24 hours';
    GET DIAGNOSTICS result_record.rows_deleted = ROW_COUNT;
    
    table_name := 'hourly_close';
    rows_deleted := result_record.rows_deleted;
    RETURN NEXT;
    
    -- Clean daily_close (7 days retention)
    DELETE FROM daily_close 
    WHERE day_ts < NOW() - INTERVAL '7 days';
    GET DIAGNOSTICS result_record.rows_deleted = ROW_COUNT;
    
    table_name := 'daily_close';
    rows_deleted := result_record.rows_deleted;
    RETURN NEXT;
    
    -- Clean weekly_close (52 weeks retention)
    DELETE FROM weekly_close 
    WHERE week_ts < NOW() - INTERVAL '52 weeks';
    GET DIAGNOSTICS result_record.rows_deleted = ROW_COUNT;
    
    table_name := 'weekly_close';
    rows_deleted := result_record.rows_deleted;
    RETURN NEXT;
    
    -- Clean old collection status (30 days retention)
    DELETE FROM collection_status 
    WHERE created_at < NOW() - INTERVAL '30 days';
    GET DIAGNOSTICS result_record.rows_deleted = ROW_COUNT;
    
    table_name := 'collection_status';
    rows_deleted := result_record.rows_deleted;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;
```

## Database Maintenance

### Statistics and Analysis

```sql
-- View for table sizes and row counts
CREATE VIEW table_stats AS
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    pg_stat_get_tuples_inserted(c.oid) as inserts,
    pg_stat_get_tuples_updated(c.oid) as updates,
    pg_stat_get_tuples_deleted(c.oid) as deletes,
    pg_stat_get_live_tuples(c.oid) as live_tuples
FROM pg_tables t
JOIN pg_class c ON c.relname = t.tablename
WHERE schemaname = 'public'
  AND tablename IN ('minute_close', 'hourly_close', 'daily_close', 'weekly_close', 'monthly_close', 'coin_priorities', 'collection_status')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Index Maintenance

```sql
-- Function to rebuild indexes if fragmentation is high
CREATE OR REPLACE FUNCTION maintain_indexes()
RETURNS TABLE(index_name TEXT, action TEXT) AS $$
DECLARE
    idx_record RECORD;
BEGIN
    FOR idx_record IN 
        SELECT indexrelname, tablename
        FROM pg_stat_user_indexes 
        WHERE schemaname = 'public'
          AND idx_tup_read > 1000
    LOOP
        -- Reindex if needed (based on your criteria)
        EXECUTE format('REINDEX INDEX CONCURRENTLY %I', idx_record.indexrelname);
        
        index_name := idx_record.indexrelname;
        action := 'reindexed';
        RETURN NEXT;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

## Migration Scripts

### Schema Version Management

```sql
-- Schema version tracking
CREATE TABLE IF NOT EXISTS schema_migrations (
    version VARCHAR(255) PRIMARY KEY,
    applied_at TIMESTAMPTZ DEFAULT NOW()
);

-- Example migration
INSERT INTO schema_migrations (version) VALUES ('001_initial_schema')
ON CONFLICT (version) DO NOTHING;
```

This database schema provides a robust foundation for cryptocurrency data storage with automatic partitioning, efficient indexing, and built-in maintenance procedures. The design balances performance, scalability, and data integrity while supporting the real-time requirements of the PulseGuard system.
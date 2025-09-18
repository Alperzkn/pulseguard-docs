# Glossary

## General Terms

**API (Application Programming Interface)**
A set of protocols, routines, and tools for building software applications. In PulseGuard, refers to the REST API service that provides access to cryptocurrency data.

**API Key**
A unique identifier used to authenticate requests to the PulseGuard API service. Required for all API endpoints except health checks.

**Collection Cycle**
A complete data gathering operation that fetches price information for all configured cryptocurrencies. Duration varies based on collection mode (4-300 minutes).

**CoinGecko**
Third-party cryptocurrency data provider that serves as PulseGuard's primary data source via their REST API.

**Data Aggregation**
The process of combining minute-level data into higher timeframes (hourly, daily, weekly, monthly) for efficient storage and querying.

**Rate Limiting**
Controlling the frequency of API requests to avoid overwhelming external services or triggering usage limits.

## Cryptocurrency Terms

**Asset ID / Coin ID**
Unique identifier for a cryptocurrency (e.g., "bitcoin", "ethereum"). Used consistently across all PulseGuard tables and APIs.

**Market Cap (Market Capitalization)**
Total value of a cryptocurrency calculated as: Current Price × Circulating Supply. Used for ranking cryptocurrencies by size.

**Market Cap Rank**
Numerical ranking of cryptocurrencies ordered by market capitalization, with 1 being the largest (typically Bitcoin).

**Price Close**
The final price recorded for a specific time period (minute, hour, day, etc.). Used as the primary price metric in PulseGuard.

**Symbol**
Short abbreviation for a cryptocurrency (e.g., BTC, ETH, ADA). May not be unique across all cryptocurrencies.

**Trading Volume (24h)**
Total dollar value of cryptocurrency traded in the last 24 hours. Indicates market activity and liquidity.

## Technical Terms

**Batch Processing**
Grouping multiple cryptocurrencies into single API requests for efficiency. PulseGuard uses batches of up to 250 coins per request.

**Collection Mode**
Configuration setting determining how many cryptocurrencies to collect:
- **top1000**: Top 1000 by market cap (~99% coverage)
- **top5000**: Top 5000 by market cap (~99.9% coverage)  
- **complete**: All available (~18,000+ coins, 100% coverage)

**Connection Pool**
Managed set of database connections shared across application processes to improve performance and resource utilization.

**Continuous Collection**
Real-time data gathering mode that collects cryptocurrency prices every minute with automatic queue management for missed data.

**CORS (Cross-Origin Resource Sharing)**
Security feature that controls which domains can access the API from web browsers. Important for dashboard functionality.

**Docker Container**
Lightweight, portable package containing an application and all its dependencies. PulseGuard uses containers for easy deployment.

**Health Check**
Automated test to verify a service is running correctly. Available at `/health` endpoint without authentication.

**Index (Database)**
Database structure that improves query performance. PulseGuard uses composite indexes on (coin_id, timestamp) combinations.

**Partitioning**
Database technique that splits large tables into smaller, more manageable pieces based on time ranges for better performance.

**Priority Coins**
Cryptocurrencies always included in collection regardless of market cap ranking (default: bitcoin, ethereum, cardano).

**Queue System**
Mechanism for managing missed data collection operations. Automatically detects and fills gaps in minute-level data.

**Retention Policy**
Rules determining how long data is kept before automatic deletion:
- Minute data: 60 minutes
- Hourly data: 24 hours  
- Daily data: 7 days
- Weekly data: 52 weeks
- Monthly data: Permanent

**WebSocket**
Communication protocol enabling real-time, bidirectional data transfer between dashboard and API service.

## Database Terms

**Aggregation Table**
Database table storing summarized data for specific timeframes (hourly_close, daily_close, etc.).

**Collection Status**
Database table tracking metadata about data collection operations including start/end times, success rates, and error messages.

**Coin Priorities**
Database table storing market cap rankings and metadata for all tracked cryptocurrencies.

**Composite Primary Key**
Database key combining multiple columns. PulseGuard uses (coin_id, timestamp) combinations for uniqueness.

**Foreign Key Constraint**
Database rule ensuring data integrity by requiring referenced records to exist in related tables.

**JSONB**
PostgreSQL data type for storing JSON documents with indexing support. Used for flexible metadata storage.

**Migration**
Script that modifies database structure (tables, indexes, constraints). PulseGuard includes numbered migration files.

**Partial Index**
Database index that only covers rows matching specific conditions. Used for frequently accessed recent data.

**UPSERT**
Database operation that inserts new records or updates existing ones. Used for handling duplicate timestamps.

## API Terms

**Authentication Header**
HTTP header containing API key for request authorization: `X-API-Key: your-key-here`

**Endpoint**
Specific URL path for accessing API functionality (e.g., `/api/v1/coins/bitcoin/history`).

**HTTP Status Code**
Numeric code indicating request result:
- 200: Success
- 401: Unauthorized (invalid API key)
- 429: Rate limited
- 500: Server error

**Query Parameter**
URL parameter for filtering or configuring API responses (e.g., `?limit=100&timeframe=hour`).

**Rate Limit**
Maximum number of API requests allowed within a time window. PulseGuard limits: 100 requests per 15 minutes.

**Response Caching**
Storing API response data temporarily to improve performance and reduce database load.

**REST (Representational State Transfer)**
Architectural style for web APIs using standard HTTP methods (GET, POST, PUT, DELETE).

**Timeframe Parameter**
API parameter specifying data granularity: `minute`, `hour`, `day`, `week`, `month`.

## Monitoring Terms

**Alert Threshold**
Configured limit that triggers notifications when exceeded (e.g., API usage > 80%, missing data > 5%).

**Collection Success Rate**
Percentage of cryptocurrencies successfully processed in a collection cycle.

**Data Completeness**
Metric indicating how much expected data was successfully collected and stored.

**Health Status**
Overall system condition indicator: `healthy`, `warning`, `error`.

**Metrics Endpoint**
API endpoint providing system performance data in Prometheus format (`/metrics`).

**Real-time Dashboard**
Web interface displaying live system status with automatic updates every 30-60 seconds.

**System Uptime**
Duration since the last system restart, indicating stability and reliability.

## Deployment Terms

**Container Registry**
Repository for storing Docker container images. Used for distributing PulseGuard applications.

**Docker Compose**
Tool for defining and running multi-container Docker applications using YAML configuration files.

**Environment Variable**
Configuration value passed to applications at runtime (e.g., `DATABASE_URL`, `API_KEY_SECRET`).

**Load Balancer**
Service distributing incoming requests across multiple application instances for reliability and performance.

**Nginx**
Web server used as reverse proxy for serving the dashboard and routing API requests.

**SSL Certificate**
Digital certificate enabling HTTPS encryption for secure web communication.

## Error Terms

**Backoff Strategy**
Technique for gradually increasing delays between retry attempts after failures (exponential backoff).

**Circuit Breaker**
Pattern that prevents cascading failures by temporarily stopping requests to failing services.

**Connection Pool Exhaustion**
Condition where all available database connections are in use, causing new requests to fail.

**Graceful Degradation**
System behavior where non-critical features are disabled while core functionality continues operating.

**Recovery Strategy**
Predefined procedures for handling and recovering from various types of system failures.

**Retry Logic**
Automated mechanism for re-attempting failed operations with configurable limits and delays.

## Performance Terms

**Caching Layer**
System component that stores frequently accessed data in memory for faster retrieval.

**Database Query Optimization**
Techniques for improving database performance through better queries, indexes, and configuration.

**Horizontal Scaling**
Adding more servers/instances to handle increased load (scale out).

**Memory Usage**
Amount of RAM consumed by system processes. Monitored to prevent out-of-memory errors.

**Response Time**
Duration between sending a request and receiving a complete response. Key performance metric.

**Vertical Scaling**
Adding more power (CPU, RAM) to existing servers to handle increased load (scale up).

---

*This glossary is regularly updated as the PulseGuard system evolves. Terms are organized alphabetically within categories for easy reference.*
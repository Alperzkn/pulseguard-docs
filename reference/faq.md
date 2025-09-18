# Frequently Asked Questions

## General Questions

### What is PulseGuard?

PulseGuard is a comprehensive cryptocurrency data collection and monitoring ecosystem consisting of three main components:
- **Data Collector**: Automated service that fetches cryptocurrency prices from CoinGecko API
- **API Service**: RESTful backend that serves collected data to applications
- **Dashboard**: Real-time web interface for monitoring system health and data quality

### What cryptocurrencies does PulseGuard support?

PulseGuard can collect data for:
- **Top 1000 mode**: Most popular 1000 cryptocurrencies (~99% market cap coverage)
- **Top 5000 mode**: Extended coverage of 5000 cryptocurrencies (~99.9% coverage)  
- **Complete mode**: All 18,000+ cryptocurrencies available on CoinGecko (100% coverage)

The system automatically prioritizes coins by market capitalization and always includes Bitcoin, Ethereum, and Cardano regardless of ranking.

### How often is data collected?

Data collection frequency depends on the mode:
- **Continuous mode**: Every minute with automatic gap filling
- **Scheduled mode**: Configurable intervals (typically every hour or daily)
- **Manual mode**: On-demand collection

Data is aggregated into multiple timeframes: minute, hourly, daily, weekly, and monthly.

### What data retention policies are used?

PulseGuard implements automatic data cleanup:
- **Minute data**: 60 minutes retention
- **Hourly data**: 24 hours retention
- **Daily data**: 7 days retention
- **Weekly data**: 52 weeks retention
- **Monthly data**: Permanent storage

This ensures optimal database performance while preserving historical trends.

## Technical Questions

### What are the system requirements?

**Minimum requirements:**
- 2 CPU cores, 4GB RAM, 20GB SSD storage
- PostgreSQL 12+, Node.js 18+
- Stable internet connection

**Recommended for production:**
- 4+ CPU cores, 8GB+ RAM, 50GB+ SSD storage
- Dedicated database server
- Load balancer for high availability

### Can I run PulseGuard on cloud platforms?

Yes, PulseGuard is designed for cloud deployment and supports:
- **Docker containers** for portable deployment
- **Kubernetes** for orchestration and scaling
- **Cloud databases** (AWS RDS, Google Cloud SQL, etc.)
- **Load balancers** for high availability
- **CDN integration** for dashboard delivery

### How does PulseGuard handle API rate limits?

PulseGuard includes intelligent rate limiting:
- **Batch processing**: Groups up to 250 coins per API request
- **Configurable delays**: Adjustable delays between requests (default: 6 seconds)
- **Exponential backoff**: Automatic retry with increasing delays for failed requests
- **Usage monitoring**: Real-time tracking of API call consumption

### What happens if the data collector fails?

PulseGuard includes robust error handling:
- **Automatic retries**: Failed requests are retried up to 3 times
- **Queue system**: Missed collections are queued for later processing
- **Gap detection**: System identifies and fills missing data automatically
- **Graceful degradation**: Falls back to priority coins if full collection fails
- **Health monitoring**: Dashboard shows collection status and alerts

## API Questions

### How do I get an API key?

API keys are configured during installation:
1. Generate a secure key: `openssl rand -hex 32`
2. Set in environment: `API_KEY_SECRET=your-generated-key`
3. Use in requests: `X-API-Key: your-generated-key`

### What are the API rate limits?

Standard rate limits:
- **100 requests per 15 minutes** for authenticated endpoints
- **100 requests per minute** for health checks
- **No limit** for system administrators

Rate limit headers are included in all responses showing remaining quota and reset time.

### Can I access historical data through the API?

Yes, the API provides comprehensive historical access:
- **Price history**: `/api/v1/coins/{id}/history` with timeframe parameter
- **Market data**: `/api/v1/market/history` for multiple coins
- **Collection logs**: `/api/v1/system/collections` for monitoring data

All endpoints support pagination and filtering by date ranges.

### How do I integrate PulseGuard with my application?

Integration options:
- **REST API**: Direct HTTP requests with API key authentication
- **JavaScript SDK**: Use the provided examples for web applications
- **Python SDK**: Sample code available for data analysis
- **WebSocket**: Real-time updates for dashboard applications

## Database Questions

### Can I use an existing PostgreSQL database?

Yes, but we recommend:
- **Dedicated database**: For optimal performance and isolation
- **Proper user permissions**: Create a dedicated user with appropriate access
- **Backup strategy**: Regular backups for data protection
- **Monitoring**: Database performance monitoring

### How much storage space is required?

Storage estimates (for top 1000 coins):
- **Minute data**: ~1GB per month
- **Hourly data**: ~50MB per month  
- **Daily data**: ~2MB per month
- **Total**: ~12GB per year including indexes and metadata

### Can I backup and restore PulseGuard data?

Yes, backup procedures are included:
- **Automated backups**: Daily PostgreSQL dumps
- **Configuration backup**: Environment files and settings
- **Log backup**: System and application logs
- **Restore procedures**: Step-by-step restoration guide

## Dashboard Questions

### Why is the dashboard not showing real-time updates?

Common causes and solutions:
1. **API connectivity**: Check if API service is running and accessible
2. **CORS configuration**: Ensure dashboard domain is allowed in API CORS settings
3. **WebSocket issues**: WebSocket connection may be blocked by firewalls
4. **API key**: Verify dashboard is configured with valid API key

### Can I customize the dashboard?

Yes, the dashboard is fully customizable:
- **Themes**: Light and dark mode support
- **Refresh intervals**: Configurable update frequencies
- **Alert thresholds**: Customizable warning and error levels
- **Component layout**: Responsive design adapts to screen size

### How do I add custom metrics?

To add custom metrics:
1. **API endpoint**: Create new endpoint in API service
2. **Data model**: Add database queries for your metrics
3. **Dashboard component**: Create React component for visualization
4. **Hook integration**: Use React Query for data fetching

## CoinGecko Questions

### What CoinGecko plan do I need?

Plan recommendations:
- **Demo plan** (30 calls/min): Suitable for top 1000 coins with 6-second delays
- **Pro plan** (500 calls/min): Recommended for top 5000 coins or faster collection
- **Enterprise plan** (1000+ calls/min): Required for complete mode or very frequent updates

### How do I upgrade my CoinGecko plan?

Signs you need to upgrade:
- Frequent rate limiting (429 errors)
- Collection taking longer than expected
- Dashboard showing API usage warnings

Upgrade process:
1. Visit CoinGecko API plans page
2. Select appropriate plan for your needs
3. Update `COINGECKO_API_KEY` with new key
4. Adjust `BATCH_DELAY_SECONDS` for higher rate limits

### What happens if CoinGecko is unavailable?

PulseGuard handles external service issues:
- **Retry logic**: Automatic retries with exponential backoff
- **Queue system**: Failed collections are queued for later retry
- **Graceful degradation**: System continues with reduced functionality
- **Monitoring**: Dashboard shows API connectivity status

## Deployment Questions

### Should I use Docker or install manually?

**Docker is recommended because:**
- Consistent environment across deployments
- Easy updates and rollbacks
- Isolated dependencies
- Simplified backup and restore
- Built-in health checks

**Manual installation when:**
- Existing infrastructure constraints
- Custom system configurations required
- Learning/development purposes

### How do I update PulseGuard?

Update procedures:
1. **Backup current data**: Database and configuration
2. **Pull latest code**: `git pull origin main`
3. **Update dependencies**: `npm install` 
4. **Run migrations**: Apply any database changes
5. **Restart services**: `docker-compose restart`
6. **Verify operation**: Check health endpoints

### Can I run multiple instances?

Yes, PulseGuard supports scaling:
- **API service**: Multiple instances behind load balancer
- **Data collector**: Single instance (avoid data duplication)
- **Dashboard**: Multiple instances for high availability
- **Database**: Read replicas for query load distribution

## Security Questions

### How secure is PulseGuard?

Security features:
- **API key authentication**: All endpoints require valid keys
- **Rate limiting**: Prevents abuse and DoS attacks
- **Input validation**: All API inputs are validated and sanitized
- **HTTPS support**: SSL/TLS encryption for all communications
- **No sensitive data exposure**: API keys and secrets are never logged

### What data is collected about users?

PulseGuard collects minimal data:
- **API usage metrics**: Request counts and response times (no personal data)
- **System logs**: Error logs and performance metrics
- **No tracking**: No cookies, analytics, or user identification

### How do I report security issues?

For security concerns:
1. **Do not** post in public issues
2. **Email** security@your-domain.com with details
3. **Include** steps to reproduce the issue
4. **Allow** reasonable time for response and fix

## Troubleshooting Questions

### Why is data collection slow?

Common causes:
- **Rate limiting**: CoinGecko API limits being hit
- **Network issues**: Slow or unreliable internet connection
- **Database performance**: Slow queries or insufficient resources
- **Configuration**: Batch size or delay settings too conservative

### How do I check system health?

Health check methods:
- **Dashboard**: Real-time status at main dashboard page
- **API endpoint**: `GET /health` (no authentication required)
- **Database**: Direct query to check recent collections
- **Logs**: Check application logs for errors

### What should I do if the API is not responding?

Troubleshooting steps:
1. **Check service status**: `docker-compose ps`
2. **View logs**: `docker-compose logs api-service`
3. **Test connectivity**: `curl http://localhost:3000/health`
4. **Restart service**: `docker-compose restart api-service`
5. **Check configuration**: Verify environment variables

## Performance Questions

### How can I improve collection performance?

Optimization strategies:
- **Upgrade CoinGecko plan**: Higher rate limits allow faster collection
- **Reduce batch delay**: Decrease `BATCH_DELAY_SECONDS` (with higher rate limits)
- **Optimize database**: Add indexes, tune PostgreSQL settings
- **Increase resources**: More CPU/RAM for data processing

### Why are API responses slow?

Performance factors:
- **Database queries**: Complex queries on large datasets
- **Missing indexes**: Database lacks proper indexes
- **Resource constraints**: Insufficient CPU/RAM
- **Network latency**: Slow connection to database

### How do I monitor performance?

Monitoring tools:
- **Built-in metrics**: Dashboard shows response times and resource usage
- **Prometheus integration**: Detailed metrics collection
- **Database monitoring**: Query performance and connection stats
- **System monitoring**: CPU, memory, and disk usage

---

*If your question isn't answered here, check the troubleshooting guide or create an issue in the project repository.*
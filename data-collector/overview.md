# PulseGuard - Cryptocurrency Data Collector

A high-performance cryptocurrency data collection system that fetches and processes price data from CoinGecko API.

## Features

- 🚀 **1000+ Cryptocurrencies** - Collects top cryptocurrencies by market cap
- 📊 **Multi-timeframe Data** - Minute, hourly, daily, weekly, monthly aggregations
- 🎯 **Market Cap Priority** - Automatic ranking and priority management
- 🔄 **Queue System** - Automatic missed minute recovery with no data gaps
- 🛡️ **Rate Limiting** - Intelligent backoff and retry logic
- 🧹 **Automatic Cleanup** - Data retention policies for each timeframe

## Quick Start

```bash
# Install dependencies
npm install

# Run complete collector once (1000 coins)
npm start

# Run continuous collector (every minute with queue system)
npm run start:continuous
```

## Project Structure

```
app/
├── index-complete.js               # Entry point for complete system
├── complete-enhanced-collector.js  # Complete collector (1000 coins)
├── continuous-collector.js         # Continuous collector with queue system
├── config.js                       # Configuration management
└── priority-manager.js             # Market cap priority system

k8s/
└── cronjob.yaml                    # Kubernetes CronJob configuration

migrations/
├── 001_schema.sql                  # Database schema
├── 002_indexes.sql                 # Performance indexes
└── 003_coin_rankings.sql           # Ranking enhancements
```

## Database Tables

- `minute_close` - Minute-by-minute price data (60 min retention)
- `hourly_close` - Hourly aggregated data (24 hour retention)
- `daily_close` - Daily aggregated data (7 day retention)  
- `weekly_close` - Weekly aggregated data (52 week retention)
- `monthly_close` - Monthly aggregated data (permanent)
- `coin_priorities` - Market cap rankings (updated daily)
- `collection_status` - Collection monitoring and metrics

## Configuration

Environment variables in `.env`:

```bash
# Collection settings
COLLECTION_MODE=top1000              # top1000 | top5000 | complete
COINS_PER_BATCH=250                  # Coins per API request
BATCH_DELAY_SECONDS=6                # Delay between batches

# CoinGecko API
COINGECKO_API_KEY=your_demo_api_key
QUOTE=usd

# Database
DATABASE_URL=postgresql://user:pass@host:5432/dbname
```

## Deployment

### DigitalOcean Droplet (Recommended)
```bash
# On your droplet
git clone your-repo && cd pulseguard
cp .env.production .env  # Edit with your credentials
./deploy-droplet.sh deploy
```

### Docker (Manual)
```bash
docker build -t crypto-collector .
docker run --env-file .env crypto-collector
```

### Kubernetes
```bash
kubectl apply -f k8s/cronjob.yaml
```

For detailed DigitalOcean deployment instructions, see [DEPLOYMENT.md](DEPLOYMENT.md).

## Data Collection Modes

- **top1000** (default): Top 1000 cryptocurrencies by market cap
- **top5000**: Extended coverage of top 5000 cryptocurrencies  
- **complete**: All available cryptocurrencies (~18,000+)
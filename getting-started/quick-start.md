# Quick Start Guide

Get PulseGuard up and running in under 10 minutes with this quick start guide.

## Prerequisites

- Node.js 18+ installed
- PostgreSQL running locally
- CoinGecko API key (free demo account)

## 1. Clone Repositories

```bash
# Create project directory
mkdir pulseguard-ecosystem && cd pulseguard-ecosystem

# Clone all components (replace with your actual repository URLs)
git clone https://github.com/your-org/pulseguard.git
git clone https://github.com/your-org/pulseguard-api.git
git clone https://github.com/your-org/pulseguard-dashboard.git
```

## 2. Database Setup

```bash
# Create database and user
sudo -u postgres psql -c "CREATE DATABASE crypto_db;"
sudo -u postgres psql -c "CREATE USER pulseguard_user WITH PASSWORD 'yourpassword';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE crypto_db TO pulseguard_user;"

# Run migrations
cd pulseguard
psql -U pulseguard_user -d crypto_db -f migrations/001_schema.sql
psql -U pulseguard_user -d crypto_db -f migrations/002_indexes.sql
psql -U pulseguard_user -d crypto_db -f migrations/003_coin_rankings.sql
```

## 3. Configure Environment

### Data Collector
```bash
cd pulseguard
cp .env.example .env
```

Edit `.env`:
```bash
DATABASE_URL=postgresql://pulseguard_user:yourpassword@localhost:5432/crypto_db
COINGECKO_API_KEY=your_demo_api_key
COLLECTION_MODE=top1000
```

### API Service
```bash
cd ../pulseguard-api
cp .env.api.example .env.api
```

Edit `.env.api`:
```bash
DATABASE_URL=postgresql://pulseguard_user:yourpassword@localhost:5432/crypto_db
API_KEY_SECRET=your-super-secret-api-key-here
PORT=3000
```

### Dashboard
```bash
cd ../pulseguard-dashboard
cp .env.example .env
```

Edit `.env`:
```bash
VITE_API_BASE_URL=http://localhost:3000/api/v1
VITE_API_KEY=your-super-secret-api-key-here
VITE_ENVIRONMENT=development
```

## 4. Install Dependencies

```bash
# Install all dependencies
cd pulseguard && npm install
cd ../pulseguard-api && npm install
cd ../pulseguard-dashboard && npm install
```

## 5. Start Services

### Terminal 1: Data Collector
```bash
cd pulseguard
npm start
```

### Terminal 2: API Service
```bash
cd pulseguard-api
npm run dev
```

### Terminal 3: Dashboard
```bash
cd pulseguard-dashboard
npm run dev
```

## 6. Verify Installation

1. **Check Data Collection**: Visit database and verify data is being collected
   ```bash
   psql -U pulseguard_user -d crypto_db -c "SELECT COUNT(*) FROM minute_close;"
   ```

2. **Test API**: Check if API responds
   ```bash
   curl -H "X-API-Key: your-super-secret-api-key-here" http://localhost:3000/api/v1/system/status
   ```

3. **Access Dashboard**: Open http://localhost:3001 in your browser

## What's Next?

- **Monitor**: Check the dashboard for real-time system monitoring
- **Customize**: Adjust collection settings in data collector configuration
- **Scale**: Deploy to production using the deployment guides
- **Explore**: Read the comprehensive documentation for advanced features

## Troubleshooting

### Common Issues

**Database Connection Failed**:
```bash
# Check PostgreSQL status
sudo systemctl status postgresql
```

**API Key Issues**:
- Verify your CoinGecko API key is valid
- Check rate limits haven't been exceeded

**Port Already in Use**:
```bash
# Find and kill processes using the ports
sudo lsof -ti:3000 | xargs kill -9
sudo lsof -ti:3001 | xargs kill -9
```

For detailed troubleshooting, see the [Installation Guide](installation.md).
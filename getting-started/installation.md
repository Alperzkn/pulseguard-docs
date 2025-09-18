# Installation Guide

This guide will help you install and set up the complete PulseGuard ecosystem on your system.

## Prerequisites

Before installing PulseGuard, ensure you have:

- **Node.js 18+** installed
- **PostgreSQL 12+** database server
- **Git** for cloning repositories
- **CoinGecko API key** (demo or paid plan)

## Quick Installation

### Option 1: Docker Setup (Recommended)

The fastest way to get PulseGuard running is using Docker:

```bash
# Clone the main repository
git clone your-pulseguard-repo pulseguard-ecosystem
cd pulseguard-ecosystem

# Start all services with Docker Compose
docker-compose up -d

# Verify installation
docker-compose ps
```

### Option 2: Manual Installation

For development or custom setups, install each component manually:

```bash
# Create project directory
mkdir pulseguard-ecosystem
cd pulseguard-ecosystem

# Clone all repositories
git clone your-data-collector-repo pulseguard
git clone your-api-service-repo pulseguard-api
git clone your-dashboard-repo pulseguard-dashboard
```

## Database Setup

### 1. PostgreSQL Installation

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**macOS:**
```bash
brew install postgresql
brew services start postgresql
```

**Windows:**
Download and install from [PostgreSQL official website](https://www.postgresql.org/download/windows/).

### 2. Database Configuration

```bash
# Connect to PostgreSQL
sudo -u postgres psql

# Create database and user
CREATE DATABASE crypto_db;
CREATE USER pulseguard_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE crypto_db TO pulseguard_user;
\q
```

### 3. Run Database Migrations

```bash
cd pulseguard
psql -U pulseguard_user -d crypto_db -f migrations/001_schema.sql
psql -U pulseguard_user -d crypto_db -f migrations/002_indexes.sql
psql -U pulseguard_user -d crypto_db -f migrations/003_coin_rankings.sql
```

## Component Installation

### 1. Data Collector Setup

```bash
cd pulseguard

# Install dependencies
npm install

# Configure environment
cp .env.example .env
nano .env
```

Edit `.env` with your configuration:
```bash
# Database connection
DATABASE_URL=postgresql://pulseguard_user:your_secure_password@localhost:5432/crypto_db

# CoinGecko API
COINGECKO_API_KEY=your_demo_api_key
COLLECTION_MODE=top1000
COINS_PER_BATCH=250
BATCH_DELAY_SECONDS=6

# Priority management
PRIORITY_COINS=bitcoin,ethereum,cardano
```

Test the installation:
```bash
# Run single collection
npm start

# Or start continuous collection
npm run start:continuous
```

### 2. API Service Setup

```bash
cd ../pulseguard-api

# Install dependencies
npm install

# Configure environment
cp .env.api.example .env.api
nano .env.api
```

Edit `.env.api`:
```bash
# Database connection (same as collector)
DATABASE_URL=postgresql://pulseguard_user:your_secure_password@localhost:5432/crypto_db

# API authentication
API_KEY_SECRET=your-super-secret-api-key-here

# Server configuration
PORT=3000
NODE_ENV=development
```

Start the API service:
```bash
# Development mode
npm run dev

# Production mode
npm start
```

Verify API is running:
```bash
curl http://localhost:3000/health
# Should return: {"status":"healthy","timestamp":"..."}
```

### 3. Dashboard Setup

```bash
cd ../pulseguard-dashboard

# Install dependencies
npm install

# Configure environment
cp .env.example .env
nano .env
```

Edit `.env`:
```bash
VITE_API_BASE_URL=http://localhost:3000/api/v1
VITE_API_KEY=your-super-secret-api-key-here
VITE_WS_URL=ws://localhost:3000/ws
VITE_ENVIRONMENT=development
```

Start the dashboard:
```bash
# Development server
npm run dev

# Production build
npm run build
npm run preview
```

Access the dashboard at `http://localhost:3001`

## Verification

### 1. Test Data Collection

```bash
# Check if data collector is working
cd pulseguard
npm start

# Verify data in database
psql -U pulseguard_user -d crypto_db -c "SELECT COUNT(*) FROM minute_close;"
```

### 2. Test API Service

```bash
# Test API endpoints
curl -H "X-API-Key: your-api-key" http://localhost:3000/api/v1/system/status
curl -H "X-API-Key: your-api-key" http://localhost:3000/api/v1/market/top/10
```

### 3. Test Dashboard

1. Open `http://localhost:3001` in your browser
2. Verify the dashboard loads without errors
3. Check that data appears in the monitoring sections
4. Confirm real-time updates are working

## Production Deployment

### Using Docker Compose

Create `docker-compose.yml` in your project root:

```yaml
version: '3.8'

services:
  database:
    image: postgres:14
    environment:
      POSTGRES_DB: crypto_db
      POSTGRES_USER: pulseguard_user
      POSTGRES_PASSWORD: your_secure_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./pulseguard/migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"

  data-collector:
    build: ./pulseguard
    depends_on:
      - database
    environment:
      - DATABASE_URL=postgresql://pulseguard_user:your_secure_password@database:5432/crypto_db
      - COINGECKO_API_KEY=your_api_key
    restart: unless-stopped

  api-service:
    build: ./pulseguard-api
    depends_on:
      - database
    environment:
      - DATABASE_URL=postgresql://pulseguard_user:your_secure_password@database:5432/crypto_db
      - API_KEY_SECRET=your-super-secret-api-key
    ports:
      - "3000:3000"
    restart: unless-stopped

  dashboard:
    build: ./pulseguard-dashboard
    ports:
      - "80:80"
    restart: unless-stopped

volumes:
  postgres_data:
```

Deploy with:
```bash
docker-compose up -d
```

### Manual Production Setup

For manual production deployment, refer to the individual deployment guides:

- [Data Collector Deployment](../data-collector/deployment.md)
- [API Service Deployment](../api-service/deployment.md)
- [Dashboard Deployment](../dashboard/deployment.md)

## Troubleshooting

### Common Issues

#### Database Connection Failed
```bash
# Check PostgreSQL status
sudo systemctl status postgresql

# Test connection manually
psql -U pulseguard_user -d crypto_db -c "SELECT version();"
```

#### API Key Issues
- Verify your CoinGecko API key is valid
- Check rate limits haven't been exceeded
- Ensure API key is correctly set in environment variables

#### Port Conflicts
```bash
# Check if ports are in use
sudo netstat -tlnp | grep :3000
sudo netstat -tlnp | grep :3001

# Kill processes using the ports if needed
sudo kill -9 $(sudo lsof -t -i:3000)
```

#### Permission Issues
```bash
# Fix npm permissions
sudo chown -R $(whoami) ~/.npm
sudo chown -R $(whoami) /usr/local/lib/node_modules
```

### Getting Help

If you encounter issues:

1. Check the logs of each component
2. Verify all environment variables are set correctly
3. Ensure database is accessible and has correct schema
4. Test API connectivity manually with curl
5. Check browser console for dashboard errors

For additional support, refer to the troubleshooting sections in each component's documentation.
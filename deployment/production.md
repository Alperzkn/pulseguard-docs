# Production Deployment Guide

## Overview

This guide covers deploying the complete PulseGuard ecosystem to a production environment using best practices for security, performance, and reliability.

## Prerequisites

- **Server**: Ubuntu 20.04+ or similar Linux distribution
- **Resources**: Minimum 4GB RAM, 2 CPU cores, 50GB SSD storage
- **Domain**: Optional custom domain with SSL certificate
- **Database**: PostgreSQL 12+ (can be same server or separate)
- **API Keys**: Valid CoinGecko API key

## Deployment Architecture

```
Internet → Nginx (SSL) → API Service (Docker) → PostgreSQL
                     ↓
            Data Collector (Docker)
                     ↓
            Dashboard (Static Files)
```

## Step 1: Server Preparation

### 1.1 Update System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip
```

### 1.2 Install Docker

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Logout and login to apply group changes
exit
```

### 1.3 Install Nginx

```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 1.4 Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Create database and user
sudo -u postgres psql <<EOF
CREATE DATABASE crypto_db;
CREATE USER pulseguard_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE crypto_db TO pulseguard_user;
\q
EOF
```

## Step 2: Application Deployment

### 2.1 Create Project Directory

```bash
sudo mkdir -p /opt/pulseguard
sudo chown $USER:$USER /opt/pulseguard
cd /opt/pulseguard
```

### 2.2 Clone Repositories

```bash
git clone https://github.com/your-org/pulseguard.git data-collector
git clone https://github.com/your-org/pulseguard-api.git api-service
git clone https://github.com/your-org/pulseguard-dashboard.git dashboard
```

### 2.3 Create Docker Compose Configuration

```yaml
# /opt/pulseguard/docker-compose.yml
version: '3.8'

services:
  data-collector:
    build: ./data-collector
    container_name: pulseguard-collector
    environment:
      - DATABASE_URL=postgresql://pulseguard_user:your_secure_password@host.docker.internal:5432/crypto_db
      - COINGECKO_API_KEY=${COINGECKO_API_KEY}
      - COLLECTION_MODE=top5000
      - BATCH_DELAY_SECONDS=2.5
      - LOG_LEVEL=warn
    volumes:
      - ./logs/collector:/app/logs
      - ./data:/app/data
    restart: unless-stopped
    depends_on:
      - database-proxy
    networks:
      - pulseguard-network

  api-service:
    build: ./api-service
    container_name: pulseguard-api
    environment:
      - DATABASE_URL=postgresql://pulseguard_user:your_secure_password@host.docker.internal:5432/crypto_db
      - API_KEY_SECRET=${API_KEY_SECRET}
      - NODE_ENV=production
      - PORT=3000
      - LOG_LEVEL=warn
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - ./logs/api:/app/logs
    restart: unless-stopped
    depends_on:
      - database-proxy
    networks:
      - pulseguard-network

  database-proxy:
    image: alpine/socat
    container_name: pulseguard-db-proxy
    command: tcp-listen:5432,fork,reuseaddr tcp-connect:host.docker.internal:5432
    ports:
      - "127.0.0.1:5433:5432"
    restart: unless-stopped
    networks:
      - pulseguard-network

networks:
  pulseguard-network:
    driver: bridge

volumes:
  postgres_data:
```

### 2.4 Environment Configuration

```bash
# Create production environment file
cat > /opt/pulseguard/.env << EOF
# CoinGecko API
COINGECKO_API_KEY=your_production_api_key

# API Authentication
API_KEY_SECRET=$(openssl rand -hex 32)

# Database
DATABASE_URL=postgresql://pulseguard_user:your_secure_password@localhost:5432/crypto_db

# Logging
LOG_LEVEL=warn
EOF

# Secure environment file
chmod 600 /opt/pulseguard/.env
```

### 2.5 Initialize Database

```bash
cd /opt/pulseguard/data-collector
psql -U pulseguard_user -d crypto_db -f migrations/001_schema.sql
psql -U pulseguard_user -d crypto_db -f migrations/002_indexes.sql
psql -U pulseguard_user -d crypto_db -f migrations/003_coin_rankings.sql
```

## Step 3: SSL Certificate Setup

### 3.1 Install Certbot

```bash
sudo apt install -y snapd
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### 3.2 Obtain SSL Certificate

```bash
# Replace your-domain.com with your actual domain
sudo certbot --nginx -d your-domain.com -d api.your-domain.com
```

## Step 4: Nginx Configuration

### 4.1 API Service Configuration

```nginx
# /etc/nginx/sites-available/pulseguard-api
server {
    listen 443 ssl http2;
    server_name api.your-domain.com;

    ssl_certificate /etc/letsencrypt/live/api.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.your-domain.com/privkey.pem;
    
    # SSL Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Health check endpoint (no rate limiting)
    location /health {
        proxy_pass http://127.0.0.1:3000/health;
        access_log off;
    }

    # API documentation
    location /docs {
        proxy_pass http://127.0.0.1:3000/docs;
    }

    # Logging
    access_log /var/log/nginx/pulseguard-api.access.log;
    error_log /var/log/nginx/pulseguard-api.error.log;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name api.your-domain.com;
    return 301 https://$server_name$request_uri;
}
```

### 4.2 Dashboard Configuration

```nginx
# /etc/nginx/sites-available/pulseguard-dashboard
server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    root /opt/pulseguard/dashboard/dist;
    index index.html;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy
    location /api/ {
        proxy_pass https://api.your-domain.com;
        proxy_ssl_verify off;
    }

    # Logging
    access_log /var/log/nginx/pulseguard-dashboard.access.log;
    error_log /var/log/nginx/pulseguard-dashboard.error.log;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}
```

### 4.3 Enable Sites

```bash
sudo ln -s /etc/nginx/sites-available/pulseguard-api /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/pulseguard-dashboard /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

## Step 5: Dashboard Build and Deployment

### 5.1 Build Dashboard

```bash
cd /opt/pulseguard/dashboard

# Install Node.js (if not already installed)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Configure environment
cat > .env.production << EOF
VITE_API_BASE_URL=https://api.your-domain.com/api/v1
VITE_API_KEY=your-api-key-here
VITE_WS_URL=wss://api.your-domain.com/ws
VITE_ENVIRONMENT=production
EOF

# Build for production
npm install
npm run build

# Set proper permissions
sudo chown -R www-data:www-data dist/
```

## Step 6: Start Services

### 6.1 Start Docker Services

```bash
cd /opt/pulseguard
docker-compose up -d

# Verify services are running
docker-compose ps
docker-compose logs -f
```

### 6.2 Verify Deployment

```bash
# Test API health
curl https://api.your-domain.com/health

# Test API with authentication
curl -H "X-API-Key: your-api-key" https://api.your-domain.com/api/v1/system/status

# Test dashboard
curl -I https://your-domain.com
```

## Step 7: Monitoring and Maintenance

### 7.1 System Monitoring

```bash
# Create monitoring script
cat > /opt/pulseguard/monitor.sh << 'EOF'
#!/bin/bash

# Check Docker containers
docker-compose ps --filter "status=running" | grep -q "Up" || {
    echo "$(date): Docker containers not running" >> /var/log/pulseguard-monitor.log
    docker-compose restart
}

# Check API health
curl -f https://api.your-domain.com/health > /dev/null 2>&1 || {
    echo "$(date): API health check failed" >> /var/log/pulseguard-monitor.log
    docker-compose restart api-service
}

# Check disk space
df -h / | awk 'NR==2 {if(substr($5,1,length($5)-1) > 85) print "$(date): Disk space low: " $5}' >> /var/log/pulseguard-monitor.log

# Check database connectivity
docker-compose exec -T api-service node -e "
const { Pool } = require('pg');
const pool = new Pool({connectionString: process.env.DATABASE_URL});
pool.query('SELECT NOW()').then(() => process.exit(0)).catch(() => process.exit(1));
" || {
    echo "$(date): Database connectivity failed" >> /var/log/pulseguard-monitor.log
}
EOF

chmod +x /opt/pulseguard/monitor.sh

# Add to crontab
(crontab -l 2>/dev/null; echo "*/5 * * * * /opt/pulseguard/monitor.sh") | crontab -
```

### 7.2 Log Rotation

```bash
# Configure logrotate
sudo cat > /etc/logrotate.d/pulseguard << EOF
/opt/pulseguard/logs/*/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 644 root root
    postrotate
        docker-compose -f /opt/pulseguard/docker-compose.yml restart data-collector api-service
    endscript
}

/var/log/nginx/pulseguard-*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 644 www-data www-data
    postrotate
        systemctl reload nginx
    endscript
}
EOF
```

### 7.3 Backup Strategy

```bash
# Create backup script
cat > /opt/pulseguard/backup.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/opt/backups/pulseguard"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Database backup
pg_dump -U pulseguard_user -h localhost crypto_db | gzip > $BACKUP_DIR/database_$DATE.sql.gz

# Configuration backup
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /opt/pulseguard/.env /opt/pulseguard/docker-compose.yml

# Application logs backup
tar -czf $BACKUP_DIR/logs_$DATE.tar.gz /opt/pulseguard/logs/

# Keep only last 7 days of backups
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete

echo "$(date): Backup completed" >> /var/log/pulseguard-backup.log
EOF

chmod +x /opt/pulseguard/backup.sh

# Schedule daily backups
(crontab -l 2>/dev/null; echo "0 2 * * * /opt/pulseguard/backup.sh") | crontab -
```

## Step 8: Security Hardening

### 8.1 Firewall Configuration

```bash
# Configure UFW firewall
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

### 8.2 Fail2Ban Setup

```bash
# Install and configure Fail2Ban
sudo apt install -y fail2ban

sudo cat > /etc/fail2ban/jail.local << EOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[nginx-http-auth]
enabled = true

[nginx-noscript]
enabled = true

[nginx-badbots]
enabled = true

[nginx-noproxy]
enabled = true
EOF

sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### 8.3 PostgreSQL Security

```bash
# Secure PostgreSQL configuration
sudo -u postgres psql << EOF
ALTER USER pulseguard_user SET default_transaction_isolation TO 'read committed';
ALTER USER pulseguard_user SET timezone TO 'UTC';
ALTER USER pulseguard_user SET client_encoding TO 'utf8';
\q
EOF

# Update PostgreSQL configuration
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = 'localhost'/" /etc/postgresql/*/main/postgresql.conf
sudo systemctl restart postgresql
```

## Step 9: Performance Optimization

### 9.1 PostgreSQL Optimization

```bash
# Calculate optimal settings based on available RAM
RAM_GB=$(free -g | awk '/^Mem:/{print $2}')
SHARED_BUFFERS=$((RAM_GB * 256))MB
EFFECTIVE_CACHE_SIZE=$((RAM_GB * 768))MB

sudo -u postgres psql << EOF
ALTER SYSTEM SET shared_buffers = '$SHARED_BUFFERS';
ALTER SYSTEM SET effective_cache_size = '$EFFECTIVE_CACHE_SIZE';
ALTER SYSTEM SET work_mem = '4MB';
ALTER SYSTEM SET maintenance_work_mem = '64MB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = 100;
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
SELECT pg_reload_conf();
\q
EOF
```

### 9.2 Nginx Optimization

```bash
# Optimize Nginx configuration
sudo sed -i 's/worker_processes auto;/worker_processes 2;/' /etc/nginx/nginx.conf
sudo sed -i 's/# worker_connections 768;/worker_connections 1024;/' /etc/nginx/nginx.conf

# Add to http block in /etc/nginx/nginx.conf
sudo sed -i '/http {/a\\n\t# Performance optimizations\n\tworker_rlimit_nofile 2048;\n\tkeepalive_timeout 30;\n\tkeepalive_requests 100;\n\tclient_max_body_size 1M;\n\tclient_body_timeout 12;\n\tclient_header_timeout 12;\n\tsend_timeout 10;' /etc/nginx/nginx.conf

sudo systemctl restart nginx
```

## Step 10: Deployment Verification

### 10.1 Health Check Script

```bash
# Create comprehensive health check
cat > /opt/pulseguard/health-check.sh << 'EOF'
#!/bin/bash

echo "=== PulseGuard Health Check ==="
echo "Date: $(date)"
echo

# Check Docker containers
echo "1. Docker Container Status:"
docker-compose ps
echo

# Check API health
echo "2. API Health Check:"
curl -s https://api.your-domain.com/health | jq . || echo "API health check failed"
echo

# Check database connectivity
echo "3. Database Connectivity:"
docker-compose exec -T api-service node -e "
const { Pool } = require('pg');
const pool = new Pool({connectionString: process.env.DATABASE_URL});
pool.query('SELECT NOW()').then(r => console.log('Database connected:', r.rows[0].now)).catch(e => console.log('Database error:', e.message));
" 2>/dev/null
echo

# Check disk space
echo "4. Disk Space:"
df -h / | grep -v Filesystem
echo

# Check memory usage
echo "5. Memory Usage:"
free -h
echo

# Check system load
echo "6. System Load:"
uptime
echo

# Check recent logs for errors
echo "7. Recent Errors in Logs:"
docker-compose logs --tail=10 data-collector | grep -i error || echo "No recent errors in data collector"
docker-compose logs --tail=10 api-service | grep -i error || echo "No recent errors in API service"
echo

echo "=== Health Check Complete ==="
EOF

chmod +x /opt/pulseguard/health-check.sh
```

### 10.2 Run Health Check

```bash
/opt/pulseguard/health-check.sh
```

## Troubleshooting

### Common Issues

1. **API not responding**: Check Docker logs and Nginx configuration
2. **Database connection errors**: Verify PostgreSQL is running and credentials are correct
3. **SSL certificate issues**: Ensure domains are correctly configured and certificates are valid
4. **High CPU usage**: Monitor collection frequency and API rate limits
5. **Disk space issues**: Implement log rotation and data cleanup

### Log Locations

- **Application logs**: `/opt/pulseguard/logs/`
- **Nginx logs**: `/var/log/nginx/`
- **System logs**: `/var/log/syslog`
- **Docker logs**: `docker-compose logs`

### Performance Monitoring

```bash
# Monitor system resources
htop

# Monitor Docker containers
docker stats

# Monitor database performance
sudo -u postgres psql -c "SELECT * FROM pg_stat_activity;"

# Monitor API response times
curl -w "@curl-format.txt" -o /dev/null -s https://api.your-domain.com/api/v1/system/status
```

This production deployment guide provides a robust, secure, and scalable setup for the PulseGuard ecosystem. Regular monitoring and maintenance will ensure optimal performance and reliability.
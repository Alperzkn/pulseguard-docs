# Docker Setup Guide

## Overview

This guide covers containerizing and deploying the PulseGuard ecosystem using Docker and Docker Compose for consistent, portable deployments across environments.

## Prerequisites

- Docker 20.10+ installed
- Docker Compose 2.0+ installed
- 4GB+ RAM available
- PostgreSQL database (local or external)

## Quick Start with Docker Compose

### 1. Complete Stack Deployment

Create a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  database:
    image: postgres:14-alpine
    container_name: pulseguard-db
    environment:
      POSTGRES_DB: crypto_db
      POSTGRES_USER: pulseguard_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./data-collector/migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pulseguard_user -d crypto_db"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Data Collector Service
  data-collector:
    build:
      context: ./data-collector
      dockerfile: Dockerfile
    container_name: pulseguard-collector
    environment:
      - DATABASE_URL=postgresql://pulseguard_user:${DB_PASSWORD}@database:5432/crypto_db
      - COINGECKO_API_KEY=${COINGECKO_API_KEY}
      - COLLECTION_MODE=${COLLECTION_MODE:-top1000}
      - BATCH_DELAY_SECONDS=${BATCH_DELAY:-6}
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - NODE_ENV=production
    volumes:
      - collector_logs:/app/logs
      - collector_data:/app/data
    depends_on:
      database:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 60s
      timeout: 30s
      retries: 3

  # API Service
  api-service:
    build:
      context: ./api-service
      dockerfile: Dockerfile
    container_name: pulseguard-api
    environment:
      - DATABASE_URL=postgresql://pulseguard_user:${DB_PASSWORD}@database:5432/crypto_db
      - API_KEY_SECRET=${API_KEY_SECRET}
      - NODE_ENV=production
      - PORT=3000
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - CORS_ORIGIN=${CORS_ORIGIN:-*}
    ports:
      - "${API_PORT:-3000}:3000"
    volumes:
      - api_logs:/app/logs
    depends_on:
      database:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Dashboard (Nginx serving static files)
  dashboard:
    build:
      context: ./dashboard
      dockerfile: Dockerfile.prod
      args:
        - VITE_API_BASE_URL=${DASHBOARD_API_URL:-http://localhost:3000/api/v1}
        - VITE_API_KEY=${API_KEY_SECRET}
    container_name: pulseguard-dashboard
    ports:
      - "${DASHBOARD_PORT:-80}:80"
    depends_on:
      - api-service
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis (Optional - for caching)
  redis:
    image: redis:7-alpine
    container_name: pulseguard-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
  collector_logs:
  collector_data:
  api_logs:
  redis_data:

networks:
  default:
    name: pulseguard-network
```

### 2. Environment Configuration

Create `.env` file:

```bash
# Database
DB_PASSWORD=your_secure_password

# API Keys
COINGECKO_API_KEY=your_coingecko_api_key
API_KEY_SECRET=your_generated_secret_key

# Collection Settings
COLLECTION_MODE=top1000
BATCH_DELAY=6
LOG_LEVEL=info

# Network Configuration
API_PORT=3000
DASHBOARD_PORT=80
CORS_ORIGIN=*

# Dashboard Configuration
DASHBOARD_API_URL=http://localhost:3000/api/v1
```

### 3. Deploy the Stack

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Check service status
docker-compose ps
```

## Individual Service Dockerfiles

### Data Collector Dockerfile

```dockerfile
# data-collector/Dockerfile
FROM node:18-alpine AS base

# Install dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Create necessary directories
RUN mkdir -p logs data && \
    chown -R nodejs:nodejs /app

USER nodejs

# Health check
COPY healthcheck.js .
HEALTHCHECK --interval=60s --timeout=30s --start-period=5s --retries=3 \
  CMD node healthcheck.js

EXPOSE 3000

CMD ["npm", "start"]
```

Health check script for data collector:

```javascript
// data-collector/healthcheck.js
const pool = require('./app/database');

async function healthCheck() {
  try {
    // Check database connection
    const result = await pool.query('SELECT NOW()');
    if (result.rows.length > 0) {
      console.log('Health check passed');
      process.exit(0);
    } else {
      throw new Error('Database query returned no results');
    }
  } catch (error) {
    console.error('Health check failed:', error.message);
    process.exit(1);
  }
}

healthCheck();
```

### API Service Dockerfile

```dockerfile
# api-service/Dockerfile
FROM node:18-alpine AS base

# Install curl for health checks
RUN apk add --no-cache curl

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    mkdir -p logs && \
    chown -R nodejs:nodejs /app

USER nodejs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["npm", "start"]
```

### Dashboard Production Dockerfile

```dockerfile
# dashboard/Dockerfile.prod
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Build arguments for environment variables
ARG VITE_API_BASE_URL
ARG VITE_API_KEY
ARG VITE_WS_URL
ARG VITE_ENVIRONMENT=production

# Set environment variables for build
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL
ENV VITE_API_KEY=$VITE_API_KEY
ENV VITE_WS_URL=$VITE_WS_URL
ENV VITE_ENVIRONMENT=$VITE_ENVIRONMENT

# Build the application
RUN npm run build

# Production stage with Nginx
FROM nginx:alpine AS production

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Copy built application
COPY --from=builder /app/dist /usr/share/nginx/html

# Add health check
RUN apk add --no-cache curl
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:80 || exit 1

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Dashboard Nginx configuration:

```nginx
# dashboard/nginx.conf
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;

    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;

        # Security headers
        add_header X-Frame-Options DENY always;
        add_header X-Content-Type-Options nosniff always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # SPA routing - serve index.html for all routes
        location / {
            try_files $uri $uri/ /index.html;
        }

        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

## Development Docker Compose

For development with hot reload:

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  database:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: crypto_db_dev
      POSTGRES_USER: pulseguard_user
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data

  data-collector:
    build:
      context: ./data-collector
      dockerfile: Dockerfile.dev
    environment:
      - DATABASE_URL=postgresql://pulseguard_user:dev_password@database:5432/crypto_db_dev
      - COINGECKO_API_KEY=${COINGECKO_API_KEY}
      - LOG_LEVEL=debug
      - NODE_ENV=development
    volumes:
      - ./data-collector:/app
      - /app/node_modules
    depends_on:
      - database
    command: npm run dev

  api-service:
    build:
      context: ./api-service
      dockerfile: Dockerfile.dev
    environment:
      - DATABASE_URL=postgresql://pulseguard_user:dev_password@database:5432/crypto_db_dev
      - API_KEY_SECRET=dev-secret-key
      - NODE_ENV=development
      - LOG_LEVEL=debug
    ports:
      - "3000:3000"
    volumes:
      - ./api-service:/app
      - /app/node_modules
    depends_on:
      - database
    command: npm run dev

  dashboard:
    build:
      context: ./dashboard
      dockerfile: Dockerfile.dev
    environment:
      - VITE_API_BASE_URL=http://localhost:3000/api/v1
      - VITE_API_KEY=dev-secret-key
    ports:
      - "3001:3001"
    volumes:
      - ./dashboard:/app
      - /app/node_modules
    depends_on:
      - api-service
    command: npm run dev

volumes:
  postgres_dev_data:
```

Development Dockerfile example:

```dockerfile
# api-service/Dockerfile.dev
FROM node:18-alpine

WORKDIR /app

# Install nodemon for development
RUN npm install -g nodemon

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev dependencies)
RUN npm install

# Copy source code
COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

## Docker Management Commands

### Basic Operations

```bash
# Build and start services
docker-compose up -d --build

# Stop services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# View logs
docker-compose logs -f [service-name]

# Execute commands in containers
docker-compose exec api-service sh
docker-compose exec database psql -U pulseguard_user crypto_db

# Scale services
docker-compose up -d --scale data-collector=2
```

### Maintenance Commands

```bash
# Update images
docker-compose pull
docker-compose up -d

# Restart specific service
docker-compose restart api-service

# View resource usage
docker stats

# Clean up unused resources
docker system prune -a

# Backup database
docker-compose exec database pg_dump -U pulseguard_user crypto_db > backup.sql

# Restore database
docker-compose exec -T database psql -U pulseguard_user crypto_db < backup.sql
```

## Production Optimizations

### Multi-stage Build Optimization

```dockerfile
# Optimized production Dockerfile
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build 2>/dev/null || echo "No build script found"

FROM node:18-alpine AS runtime
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
WORKDIR /app

# Copy production dependencies
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules

# Copy application code
COPY --chown=nodejs:nodejs . .

# Copy built assets if they exist
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist 2>/dev/null || true

USER nodejs

EXPOSE 3000
CMD ["npm", "start"]
```

### Resource Limits

```yaml
# docker-compose.yml with resource limits
services:
  data-collector:
    # ... other configuration
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'

  api-service:
    # ... other configuration
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'

  database:
    # ... other configuration
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'
```

## Monitoring and Logging

### Centralized Logging

```yaml
# Add to docker-compose.yml
services:
  # ... other services

  # Centralized logging
  fluentd:
    image: fluent/fluentd:v1.14-debian-1
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - ./logs:/var/log
    ports:
      - "24224:24224"

# Update services to use fluentd logging
  api-service:
    # ... other configuration
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: api-service
```

### Health Monitoring

```bash
# Create monitoring script
cat > monitor-docker.sh << 'EOF'
#!/bin/bash

echo "=== Docker Services Health Check ==="
echo "Timestamp: $(date)"
echo

# Check container status
docker-compose ps

echo -e "\n=== Service Health Checks ==="

# Check each service health
for service in data-collector api-service dashboard database; do
    health=$(docker-compose ps -q $service | xargs docker inspect --format='{{.State.Health.Status}}' 2>/dev/null)
    if [ "$health" = "healthy" ]; then
        echo "✅ $service: healthy"
    else
        echo "❌ $service: $health"
    fi
done

echo -e "\n=== Resource Usage ==="
docker stats --no-stream

echo -e "\n=== Recent Logs (Errors) ==="
docker-compose logs --tail=10 | grep -i error || echo "No recent errors found"
EOF

chmod +x monitor-docker.sh
```

## Troubleshooting

### Common Issues

1. **Port conflicts**: Change ports in docker-compose.yml
2. **Memory issues**: Increase Docker memory allocation
3. **Database connection**: Check network connectivity between containers
4. **Permission issues**: Ensure proper user permissions in Dockerfiles
5. **Build failures**: Check Dockerfile syntax and build context

### Debugging Commands

```bash
# Debug container issues
docker-compose logs [service-name]
docker-compose exec [service-name] sh
docker inspect [container-name]

# Check container resource usage
docker stats [container-name]

# Network debugging
docker network ls
docker network inspect pulseguard-network

# Volume debugging
docker volume ls
docker volume inspect [volume-name]
```

### Performance Tuning

```bash
# Optimize Docker daemon
cat > /etc/docker/daemon.json << EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

systemctl restart docker
```

This Docker setup provides a robust, scalable foundation for deploying PulseGuard in any environment with consistent behavior and easy management.
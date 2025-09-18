# PulseGuard System Architecture

## System Overview

PulseGuard is designed as a modular cryptocurrency data pipeline with three main components that work together to collect, serve, and visualize market data.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    PulseGuard Ecosystem                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │   Data          │    │   API Service   │    │   Dashboard     │ │
│  │   Collector     │    │   (Backend)     │    │   (Frontend)    │ │
│  │                 │    │                 │    │                 │ │
│  │ - Node.js       │───▶│ - Express.js    │───▶│ - React + TS    │ │
│  │ - CoinGecko API │    │ - PostgreSQL    │    │ - Tailwind CSS  │ │
│  │ - Cron Jobs     │    │ - Authentication│    │ - React Query   │ │
│  │ - Rate Limiting │    │ - Rate Limiting │    │ - WebSocket     │ │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘ │
│           │                       │                       │       │
│           ▼                       ▼                       ▼       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │              PostgreSQL Database                        │       │
│  │ - Time-series tables  - System metrics                 │       │
│  │ - Market cap data     - Collection logs                │       │
│  │ - Performance indexes - Health monitoring              │       │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Component Architecture

### 1. Data Collector
**Purpose**: Automated cryptocurrency data collection and processing

```
┌─────────────────────────────────────────────────────────────┐
│                    Data Collector                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │  Priority       │    │  Collection     │    │  Data       │ │
│  │  Manager        │    │  Engine         │    │  Processor  │ │
│  │                 │    │                 │    │             │ │
│  │ - Market Cap    │───▶│ - Batch Proc.   │───▶│ - Validation│ │
│  │ - Coin Rankings │    │ - Rate Limiting │    │ - Transform │ │
│  │ - Cache Mgmt    │    │ - Error Handling│    │ - Storage   │ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
│           │                       │                       │   │
│           ▼                       ▼                       ▼   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │  Configuration  │    │  CoinGecko API  │    │ PostgreSQL  │ │
│  │  Management     │    │  Integration    │    │ Database    │ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Key Features**:
- Market cap priority system
- Intelligent rate limiting
- Automatic data aggregation
- Queue-based missed minute recovery

### 2. API Service
**Purpose**: RESTful backend for serving collected data

```
┌─────────────────────────────────────────────────────────────┐
│                     API Service                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │  Authentication │    │  Rate Limiter   │    │  Request    │ │
│  │  Middleware     │    │  Middleware     │    │  Validator  │ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
│           │                       │                       │   │
│           ▼                       ▼                       ▼   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Route Handlers                       │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │   │
│  │  │   Coins     │ │   Market    │ │   System    │       │   │
│  │  │ Controller  │ │ Controller  │ │ Controller  │       │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘       │   │
│  └─────────────────────────────────────────────────────────────┘ │
│           │                       │                       │   │
│           ▼                       ▼                       ▼   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Data Models                          │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │   │
│  │  │ Coins Model │ │Market Model │ │System Model │       │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘       │   │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Key Features**:
- API key authentication
- Request rate limiting
- Comprehensive error handling
- Health monitoring endpoints

### 3. Dashboard
**Purpose**: Real-time monitoring and visualization interface

```
┌─────────────────────────────────────────────────────────────┐
│                     Dashboard                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │     Pages       │    │   Components    │    │    Hooks    │ │
│  │                 │    │                 │    │             │ │
│  │ - Dashboard     │    │ - Charts        │    │ - API Calls │ │
│  │ - System Status │    │ - Tables        │    │ - WebSocket │ │
│  │ - Data Monitor  │    │ - Alerts        │    │ - State Mgmt│ │
│  │ - API Usage     │    │ - Cards         │    │             │ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
│           │                       │                       │   │
│           ▼                       ▼                       ▼   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   State Management                     │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │   │
│  │  │   Zustand   │ │TanStack Query│ │  WebSocket  │       │   │
│  │  │ (App State) │ │(Server State)│ │(Real-time)  │       │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘       │   │
│  └─────────────────────────────────────────────────────────────┘ │
│           │                       │                       │   │
│           ▼                       ▼                       ▼   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    API Services                        │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │   │
│  │  │System Service│ │Market Service│ │WebSocket Svc│      │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘       │   │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Key Features**:
- Real-time data visualization
- System health monitoring
- Interactive charts and tables
- Responsive design

## Data Flow Architecture

### 1. Data Collection Flow
```
CoinGecko API → Rate Limiter → Batch Processor → Data Validator → PostgreSQL
      ↓                                                              ↓
Priority Cache ←─── Market Cap Rankings ←─── Daily Priority Refresh ─┘
```

### 2. API Service Flow
```
Client Request → Auth Middleware → Rate Limiter → Route Handler → Data Model → PostgreSQL
       ↓                                             ↓
Error Handler ←─── Response Formatter ←─── Business Logic ←─────────────┘
```

### 3. Dashboard Data Flow
```
User Interaction → React Component → Custom Hook → API Client → PulseGuard API
       ↓                    ↓              ↓            ↓
   UI Update ←─── State Update ←─── TanStack Query ←─── Response Cache
```

### 4. Real-time Updates
```
Data Collector → Database Trigger → WebSocket Server → Dashboard Client → UI Update
```

## Database Architecture

### Table Structure
```
Time-Series Tables (Partitioned by time):
├── minute_close    (60 min retention)
├── hourly_close    (24 hour retention)
├── daily_close     (7 day retention)
├── weekly_close    (52 week retention)
└── monthly_close   (permanent retention)

Management Tables:
├── coin_priorities    (market cap rankings)
├── collection_status  (system health)
└── api_usage_stats   (monitoring data)
```

### Data Relationships
```
coin_priorities (1) ──── (N) minute_close
       │                        │
       │                        ├── hourly_close
       │                        ├── daily_close
       │                        ├── weekly_close
       │                        └── monthly_close
       │
collection_status (1) ──── (N) batch_logs
```

## Security Architecture

### Authentication Flow
```
Client Request → API Key Validation → Rate Limit Check → Resource Access
       ↓                    ↓                  ↓               ↓
   Reject (401) ←─── Invalid Key ←─── Rate Limited (429) ←─── Success (200)
```

### Network Security
```
Internet → Firewall → Load Balancer → API Service → Database
    ↓                     ↓               ↓            ↓
   HTTPS               SSL Term.      Internal Net   Private Net
```

## Deployment Architecture

### Development Environment
```
Developer Machine:
├── Node.js (Data Collector)
├── Express.js (API Service)
├── React Dev Server (Dashboard)
└── Local PostgreSQL
```

### Production Environment
```
Production Server:
├── Docker Container (Data Collector)
├── Docker Container (API Service)
├── Static Files (Dashboard)
├── PostgreSQL Database
├── Nginx (Reverse Proxy)
└── SSL Certificates
```

### Container Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                Production Deployment                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │     Nginx       │    │    Docker       │    │ PostgreSQL  │ │
│  │  (Reverse Proxy)│───▶│   Containers    │───▶│  Database   │ │
│  │                 │    │                 │    │             │ │
│  │ - SSL Term.     │    │ - Collector     │    │ - Data      │ │
│  │ - Load Balance  │    │ - API Service   │    │ - Backups   │ │
│  │ - Static Files  │    │ - Health Checks │    │ - Monitoring│ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Monitoring Architecture

### System Monitoring
```
Application Logs → Log Aggregator → Monitoring Dashboard
       ↓                ↓                    ↓
   Error Alerts ←─── Threshold Check ←─── Health Metrics
```

### Performance Monitoring
```
Response Times → Metrics Collector → Performance Dashboard
       ↓               ↓                     ↓
   SLA Alerts ←─── Trend Analysis ←─── Historical Data
```

## Scalability Considerations

### Horizontal Scaling
- **API Service**: Multiple instances behind load balancer
- **Data Collector**: Distributed collection across regions
- **Database**: Read replicas for query load distribution

### Vertical Scaling
- **CPU**: Increased processing power for concurrent operations
- **Memory**: Larger caches for improved response times
- **Storage**: Faster SSDs for database performance

### Future Architecture
```
Multi-Region Deployment:
├── Global Load Balancer
├── Regional API Clusters
├── Distributed Database (Primary/Replica)
├── CDN for Dashboard Assets
└── Centralized Monitoring
```
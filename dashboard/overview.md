# PulseGuard Dashboard

A modern React dashboard for monitoring PulseGuard cryptocurrency data collection system with HTTPS support.

## Features

### Phase 1: Core Infrastructure ✅ COMPLETED
- React + TypeScript + Vite setup
- Tailwind CSS styling
- React Router for navigation
- TanStack Query for API state management
- Basic layout and routing

### Phase 2: Monitoring Features ✅ COMPLETED
- **Data Completeness Dashboard**: Real-time tracking of coin collection across all timeframes
- **API Usage Monitor**: CoinGecko rate limit tracking and upgrade recommendations
- **Missing Coin Detection**: Real-time identification of failed collections
- **Services Health**: System monitoring and resource usage
- **Database Health**: PostgreSQL table statistics and status

### Phase 3: Advanced Features (Coming Soon)
- Market data visualization
- Interactive charts
- Performance metrics
- Alert system

### Phase 4: Configuration (Coming Soon)
- Real-time WebSocket integration
- Configuration panel
- User management
- Export features

## Key Components

### Data Completeness Monitoring
The dashboard tracks exactly how many coins are fetched every minute across all database tables:
- **Real-time tracking**: Monitor 1000-coin collection target
- **Missing coin identification**: See which specific coins failed to collect
- **API limit detection**: Track when CoinGecko demo account limits are hit
- **Upgrade recommendations**: Know when to upgrade to paid CoinGecko plans

### API Integration
- **PulseGuard API**: Full integration with your existing API service
- **Fallback handling**: Graceful degradation when new endpoints aren't available
- **Type-safe**: Full TypeScript integration

## Development

### Prerequisites
- Node.js 18+
- Access to PulseGuard API at http://164.92.238.238:3000

### Local Development
```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Visit http://localhost:3001
```

### Environment Variables
```bash
VITE_API_BASE_URL=http://164.92.238.238:3000/api/v1
VITE_API_KEY=your-api-key
VITE_WS_URL=ws://164.92.238.238:3000/ws
VITE_ENVIRONMENT=development
```

### Build for Production
```bash
npm run build
npm run preview
```

## Project Structure
```
src/
├── components/          # Reusable UI components
│   ├── layout/         # Layout components (Header, Sidebar)
│   ├── monitoring/     # Data monitoring components
│   └── ui/             # Basic UI components
├── pages/              # Dashboard pages
├── services/           # API integration
├── types/              # TypeScript type definitions
├── hooks/              # Custom React hooks
└── utils/              # Utility functions
```

## API Integration

The dashboard integrates with your PulseGuard API service and includes fallback handling for new endpoints that haven't been implemented yet. Key endpoints:

- `GET /system/status` - System health and collection status
- `GET /system/collections` - Collection history
- `GET /system/database` - Database statistics
- `GET /system/completeness` - Data completeness monitoring (fallback included)
- `GET /system/missing-coins` - Missing coin detection (fallback included)

## Monitoring Capabilities

1. **1000-Coin Target Tracking**: Monitor exactly how many coins are collected per minute/hour/day
2. **API Limit Detection**: Know when CoinGecko rate limits are being hit  
3. **Upgrade Recommendations**: Get specific guidance on when to upgrade CoinGecko plans
4. **Missing Coin Alerts**: See which coins failed to collect with specific coin IDs
5. **Success Rate Monitoring**: Track collection success rates across all timeframes
6. **Real-time Updates**: Automatic refresh every 30-60 seconds

This dashboard directly addresses your need to track coin collection completeness and determine when to upgrade from the CoinGecko demo account.

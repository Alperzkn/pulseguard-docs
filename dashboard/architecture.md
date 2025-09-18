# PulseGuard Dashboard Architecture

## System Overview

The PulseGuard Dashboard is a modern React-based Single Page Application (SPA) that provides real-time monitoring and visualization for the PulseGuard cryptocurrency data collection ecosystem.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    PulseGuard Dashboard                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │   Frontend      │    │   API Layer     │    │   Data Layer    │ │
│  │   (React SPA)   │    │   (REST API)    │    │   (PostgreSQL)  │ │
│  │                 │    │                 │    │                 │ │
│  │ - TypeScript    │───▶│ - Express.js    │───▶│ - Time Series   │ │
│  │ - Tailwind CSS  │    │ - Authentication│    │ - Market Data   │ │
│  │ - React Query   │    │ - Rate Limiting │    │ - System Logs   │ │
│  │ - Zustand       │    │ - Error Handler │    │ - Health Metrics│ │
│  │ - Recharts      │    │ - CORS Support  │    │                 │ │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘ │
│           │                       │                       │       │
│           ▼                       ▼                       ▼       │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │   User Browser  │    │   Data Collector│    │   CoinGecko API │ │
│  │   (Client)      │    │   (Cron Service)│    │   (External)    │ │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Frontend Architecture

### 1. Component Hierarchy

```
App.tsx
├── Layout/
│   ├── Header.tsx                 # Main navigation bar
│   ├── Sidebar.tsx               # Navigation menu
│   └── Footer.tsx                # Status bar
├── Pages/
│   ├── Dashboard.tsx             # Main overview page
│   ├── SystemStatus.tsx          # System health monitoring
│   ├── DataCompleteness.tsx      # Data quality monitoring
│   ├── ApiUsage.tsx              # CoinGecko API usage tracking
│   ├── MarketOverview.tsx        # Market data visualization
│   └── Settings.tsx              # Configuration panel
├── Components/
│   ├── monitoring/
│   │   ├── SystemMetrics.tsx     # Live system statistics
│   │   ├── CollectionStatus.tsx  # Data collection status
│   │   ├── DatabaseHealth.tsx    # Database connection status
│   │   └── AlertPanel.tsx        # Alert and notification system
│   ├── charts/
│   │   ├── PriceChart.tsx        # Cryptocurrency price charts
│   │   ├── VolumeChart.tsx       # Trading volume visualization
│   │   └── StatusChart.tsx       # System status over time
│   └── ui/
│       ├── Card.tsx              # Reusable card component
│       ├── Table.tsx             # Data table component
│       ├── Modal.tsx             # Modal dialog component
│       └── LoadingSpinner.tsx    # Loading state component
└── hooks/
    ├── useSystemStatus.ts        # System status data hook
    ├── useMarketData.ts          # Market data fetching hook
    ├── useApiUsage.ts            # API usage monitoring hook
    └── useWebSocket.ts           # Real-time data connection
```

### 2. State Management

#### Global State (Zustand)
```typescript
interface AppState {
  // System state
  connectionStatus: 'connected' | 'disconnected' | 'connecting';
  lastUpdate: Date;
  
  // User preferences
  theme: 'light' | 'dark';
  refreshInterval: number;
  notifications: boolean;
  
  // Dashboard configuration
  selectedTimeframe: 'minute' | 'hour' | 'day' | 'week' | 'month';
  selectedCoins: string[];
  alertThresholds: AlertThresholds;
}
```

#### Server State (TanStack Query)
```typescript
const queryKeys = {
  systemStatus: ['system', 'status'],
  marketData: ['market', 'data'],
  apiUsage: ['api', 'usage'],
  collections: ['system', 'collections'],
  databaseHealth: ['system', 'database'],
  completeness: ['system', 'completeness']
};
```

### 3. Data Flow

```
User Interaction → Component → Custom Hook → API Call → TanStack Query → Component Update → UI Render
                                     ↓
Real-time Updates ← WebSocket ← API Server ← Database ← Data Collector
```

## API Integration Layer

### 1. Service Architecture

```typescript
// src/services/
├── api.ts                        # Base API client
├── systemService.ts              # System monitoring endpoints
├── marketService.ts              # Market data endpoints
├── authService.ts                # Authentication handling
└── websocketService.ts           # Real-time data connection
```

### 2. API Client Configuration

```typescript
class ApiClient {
  private baseURL: string;
  private apiKey: string;
  private timeout: number = 10000;
  
  constructor(config: ApiConfig) {
    this.baseURL = config.baseURL;
    this.apiKey = config.apiKey;
  }
  
  // Request interceptor for authentication
  private addAuthHeader(config: RequestConfig): RequestConfig {
    return {
      ...config,
      headers: {
        ...config.headers,
        'X-API-Key': this.apiKey,
        'Content-Type': 'application/json'
      }
    };
  }
  
  // Response interceptor for error handling
  private handleResponse<T>(response: Response): Promise<ApiResponse<T>> {
    if (!response.ok) {
      throw new ApiError(response.status, response.statusText);
    }
    return response.json();
  }
}
```

### 3. Error Handling Strategy

```typescript
interface ErrorHandling {
  // Network errors
  connectionError: () => void;
  timeoutError: () => void;
  
  // API errors
  authenticationError: () => void;
  rateLimitError: () => void;
  serverError: () => void;
  
  // Data errors
  missingDataError: () => void;
  invalidDataError: () => void;
}
```

## Real-time Features

### 1. WebSocket Connection

```typescript
class WebSocketManager {
  private connection: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  
  connect(): void {
    this.connection = new WebSocket(config.wsUrl);
    
    this.connection.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handleRealTimeUpdate(data);
    };
    
    this.connection.onclose = () => {
      this.attemptReconnect();
    };
  }
  
  private handleRealTimeUpdate(data: RealtimeData): void {
    // Update relevant components
    queryClient.setQueryData(['system', 'status'], data.systemStatus);
    queryClient.setQueryData(['market', 'data'], data.marketData);
  }
}
```

### 2. Auto-refresh Strategy

```typescript
const useAutoRefresh = (key: QueryKey, interval: number) => {
  return useQuery({
    queryKey: key,
    queryFn: () => fetchData(key),
    refetchInterval: interval,
    refetchIntervalInBackground: false,
    staleTime: interval / 2,
    cacheTime: interval * 2
  });
};
```

## Performance Optimization

### 1. Code Splitting

```typescript
// Lazy loading for route components
const Dashboard = lazy(() => import('./pages/Dashboard'));
const SystemStatus = lazy(() => import('./pages/SystemStatus'));
const DataCompleteness = lazy(() => import('./pages/DataCompleteness'));

// Bundle splitting configuration
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          charts: ['recharts'],
          state: ['@tanstack/react-query', 'zustand']
        }
      }
    }
  }
});
```

### 2. Memoization Strategy

```typescript
// Component memoization
const SystemMetrics = memo(({ data }: SystemMetricsProps) => {
  const chartData = useMemo(() => 
    processMetricsData(data), [data]
  );
  
  return <Chart data={chartData} />;
});

// Query optimization
const useOptimizedMarketData = (coinIds: string[]) => {
  return useQuery({
    queryKey: ['market', 'data', coinIds.sort().join(',')],
    queryFn: () => fetchMarketData(coinIds),
    select: useCallback(
      (data) => data.filter(coin => coinIds.includes(coin.id)),
      [coinIds]
    )
  });
};
```

### 3. Virtual Scrolling

```typescript
// For large datasets
const VirtualizedTable = ({ data }: { data: CoinData[] }) => {
  return (
    <FixedSizeList
      height={600}
      itemCount={data.length}
      itemSize={50}
      itemData={data}
    >
      {CoinRow}
    </FixedSizeList>
  );
};
```

## Security Implementation

### 1. API Key Management

```typescript
// Environment variable validation
const validateConfig = (): Config => {
  const apiKey = import.meta.env.VITE_API_KEY;
  const apiUrl = import.meta.env.VITE_API_BASE_URL;
  
  if (!apiKey || !apiUrl) {
    throw new Error('Missing required environment variables');
  }
  
  return { apiKey, apiUrl };
};
```

### 2. Input Sanitization

```typescript
const sanitizeInput = (input: string): string => {
  return input
    .replace(/[<>]/g, '') // Remove HTML tags
    .replace(/[&"']/g, match => { // Escape entities
      const entities: Record<string, string> = {
        '&': '&amp;',
        '"': '&quot;',
        "'": '&#x27;'
      };
      return entities[match];
    });
};
```

### 3. Content Security Policy

```typescript
// vite.config.ts
export default defineConfig({
  plugins: [
    react(),
    {
      name: 'csp',
      configureServer(server) {
        server.middlewares.use((_req, res, next) => {
          res.setHeader(
            'Content-Security-Policy',
            "default-src 'self'; connect-src 'self' ws: wss:; style-src 'self' 'unsafe-inline'"
          );
          next();
        });
      }
    }
  ]
});
```

## Testing Strategy

### 1. Unit Testing

```typescript
// Component testing
test('SystemMetrics displays correct data', () => {
  const mockData = { cpu: 45, memory: 70, disk: 30 };
  render(<SystemMetrics data={mockData} />);
  
  expect(screen.getByText('45%')).toBeInTheDocument();
  expect(screen.getByText('70%')).toBeInTheDocument();
  expect(screen.getByText('30%')).toBeInTheDocument();
});
```

### 2. Integration Testing

```typescript
// API integration testing
test('fetches and displays system status', async () => {
  server.use(
    rest.get('/api/v1/system/status', (req, res, ctx) => {
      return res(ctx.json({ status: 'healthy', uptime: 3600 }));
    })
  );
  
  render(<Dashboard />);
  
  expect(await screen.findByText('healthy')).toBeInTheDocument();
});
```

### 3. E2E Testing

```typescript
// Playwright/Cypress tests
test('dashboard navigation works correctly', async ({ page }) => {
  await page.goto('/');
  await page.click('[data-testid="system-status-link"]');
  await expect(page).toHaveURL('/system-status');
});
```

## Deployment Architecture

### 1. Build Process

```bash
# Production build pipeline
npm run build
├── Static asset optimization
├── Code splitting and chunking
├── Environment variable injection
├── Source map generation
└── Bundle analysis
```

### 2. CDN Integration

```typescript
// Asset delivery optimization
const cdnConfig = {
  staticAssets: 'https://cdn.pulseguard.com/assets/',
  apiEndpoint: 'https://api.pulseguard.com/v1/',
  wsEndpoint: 'wss://api.pulseguard.com/ws'
};
```

---

*Last updated: 2025-09-18*
*Version: 1.0*
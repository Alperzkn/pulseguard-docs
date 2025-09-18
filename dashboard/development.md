# Dashboard Development Guide

## Development Environment Setup

### Prerequisites

- Node.js 18+ installed
- PulseGuard API service running
- Code editor (VS Code recommended)
- Browser with React DevTools extension

### Local Development Setup

1. **Clone and Install**
   ```bash
   git clone your-dashboard-repo pulseguard-dashboard
   cd pulseguard-dashboard
   npm install
   ```

2. **Environment Configuration**
   ```bash
   cp .env.example .env
   ```

   Configure `.env`:
   ```bash
   VITE_API_BASE_URL=http://localhost:3000/api/v1
   VITE_API_KEY=your-development-api-key
   VITE_WS_URL=ws://localhost:3000/ws
   VITE_ENVIRONMENT=development
   VITE_DEBUG=true
   ```

3. **Start Development Server**
   ```bash
   npm run dev
   # Dashboard available at http://localhost:3001
   ```

## Project Structure Deep Dive

```
src/
├── components/            # Reusable UI components
│   ├── layout/           # Layout components
│   │   ├── Header.tsx    # Main navigation header
│   │   ├── Sidebar.tsx   # Navigation sidebar
│   │   └── Layout.tsx    # Main layout wrapper
│   ├── monitoring/       # Monitoring specific components
│   │   ├── SystemMetrics.tsx        # System performance metrics
│   │   ├── CollectionStatus.tsx     # Data collection status
│   │   ├── DatabaseHealth.tsx       # Database monitoring
│   │   ├── ApiUsageMonitor.tsx      # API usage tracking
│   │   └── AlertPanel.tsx           # Alert notifications
│   ├── charts/           # Chart components
│   │   ├── LineChart.tsx            # Line chart component
│   │   ├── AreaChart.tsx            # Area chart component
│   │   ├── BarChart.tsx             # Bar chart component
│   │   └── StatusChart.tsx          # Status visualization
│   └── ui/               # Basic UI components
│       ├── Card.tsx      # Card container
│       ├── Button.tsx    # Button component
│       ├── Modal.tsx     # Modal dialog
│       ├── Table.tsx     # Data table
│       ├── Badge.tsx     # Status badges
│       └── Spinner.tsx   # Loading spinner
├── pages/                # Page components
│   ├── Dashboard.tsx     # Main dashboard page
│   ├── SystemStatus.tsx  # System monitoring page
│   ├── DataCompleteness.tsx  # Data quality monitoring
│   ├── ApiUsage.tsx      # API usage analytics
│   ├── MarketOverview.tsx     # Market data visualization
│   └── Settings.tsx      # Configuration settings
├── hooks/                # Custom React hooks
│   ├── useSystemStatus.ts     # System status data
│   ├── useMarketData.ts       # Market data fetching
│   ├── useApiUsage.ts         # API usage monitoring
│   ├── useWebSocket.ts        # WebSocket connection
│   ├── useLocalStorage.ts     # Local storage hook
│   └── useDebounce.ts         # Debounced values
├── services/             # API and external services
│   ├── api.ts            # Base API client
│   ├── systemService.ts  # System monitoring API
│   ├── marketService.ts  # Market data API
│   ├── websocketService.ts    # WebSocket service
│   └── config.ts         # Configuration management
├── types/                # TypeScript type definitions
│   ├── api.ts            # API response types
│   ├── system.ts         # System monitoring types
│   ├── market.ts         # Market data types
│   └── common.ts         # Common shared types
├── contexts/             # React contexts
│   ├── ThemeContext.tsx  # Theme management
│   ├── AuthContext.tsx   # Authentication context
│   └── AlertContext.tsx  # Alert notifications
├── utils/                # Utility functions
│   ├── formatters.ts     # Data formatting utilities
│   ├── validators.ts     # Input validation
│   ├── constants.ts      # Application constants
│   └── helpers.ts        # General helper functions
└── styles/               # Styling files
    ├── globals.css       # Global styles
    ├── components.css    # Component-specific styles
    └── themes.css        # Theme variables
```

## Component Development

### 1. Creating Reusable Components

#### Base Component Template

```typescript
// src/components/ui/Card.tsx
import React from 'react';
import { cn } from '@/utils/helpers';

interface CardProps extends React.HTMLAttributes<HTMLDivElement> {
  children: React.ReactNode;
  variant?: 'default' | 'outlined' | 'elevated';
  size?: 'sm' | 'md' | 'lg';
}

export const Card: React.FC<CardProps> = ({ 
  children, 
  variant = 'default', 
  size = 'md',
  className,
  ...props 
}) => {
  const variants = {
    default: 'bg-white border border-gray-200',
    outlined: 'bg-transparent border-2 border-gray-300',
    elevated: 'bg-white shadow-lg border-0'
  };

  const sizes = {
    sm: 'p-3',
    md: 'p-4',
    lg: 'p-6'
  };

  return (
    <div 
      className={cn(
        'rounded-lg',
        variants[variant],
        sizes[size],
        className
      )}
      {...props}
    >
      {children}
    </div>
  );
};
```

#### Monitoring Component Example

```typescript
// src/components/monitoring/SystemMetrics.tsx
import React from 'react';
import { useSystemStatus } from '@/hooks/useSystemStatus';
import { Card } from '@/components/ui/Card';
import { Badge } from '@/components/ui/Badge';
import { LineChart } from '@/components/charts/LineChart';

interface SystemMetricsProps {
  refreshInterval?: number;
  showCharts?: boolean;
}

export const SystemMetrics: React.FC<SystemMetricsProps> = ({
  refreshInterval = 30000,
  showCharts = true
}) => {
  const { data: systemStatus, isLoading, error } = useSystemStatus({
    refetchInterval: refreshInterval
  });

  if (isLoading) {
    return (
      <Card>
        <div className="animate-pulse space-y-4">
          <div className="h-4 bg-gray-200 rounded w-1/4"></div>
          <div className="h-8 bg-gray-200 rounded"></div>
        </div>
      </Card>
    );
  }

  if (error) {
    return (
      <Card variant="outlined">
        <div className="text-red-600">
          <h3 className="font-semibold">Error Loading System Metrics</h3>
          <p className="text-sm">{error.message}</p>
        </div>
      </Card>
    );
  }

  const getStatusColor = (status: string) => {
    switch (status) {
      case 'healthy': return 'green';
      case 'warning': return 'yellow';
      case 'error': return 'red';
      default: return 'gray';
    }
  };

  return (
    <Card>
      <div className="space-y-4">
        <div className="flex items-center justify-between">
          <h3 className="text-lg font-semibold">System Status</h3>
          <Badge color={getStatusColor(systemStatus?.status)}>
            {systemStatus?.status?.toUpperCase()}
          </Badge>
        </div>

        <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
          <div className="text-center">
            <div className="text-2xl font-bold text-blue-600">
              {systemStatus?.uptime ? formatUptime(systemStatus.uptime) : '--'}
            </div>
            <div className="text-sm text-gray-500">Uptime</div>
          </div>

          <div className="text-center">
            <div className="text-2xl font-bold text-green-600">
              {systemStatus?.collectionsToday || 0}
            </div>
            <div className="text-sm text-gray-500">Collections Today</div>
          </div>

          <div className="text-center">
            <div className="text-2xl font-bold text-purple-600">
              {systemStatus?.successRate?.toFixed(1) || 0}%
            </div>
            <div className="text-sm text-gray-500">Success Rate</div>
          </div>

          <div className="text-center">
            <div className="text-2xl font-bold text-orange-600">
              {systemStatus?.apiUsage?.callsToday || 0}
            </div>
            <div className="text-sm text-gray-500">API Calls</div>
          </div>
        </div>

        {showCharts && systemStatus?.metrics && (
          <div className="mt-6">
            <h4 className="text-md font-medium mb-3">Performance Trends</h4>
            <LineChart
              data={systemStatus.metrics}
              xField="timestamp"
              yField="value"
              height={200}
            />
          </div>
        )}
      </div>
    </Card>
  );
};

// Helper function
const formatUptime = (seconds: number): string => {
  const days = Math.floor(seconds / 86400);
  const hours = Math.floor((seconds % 86400) / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  
  if (days > 0) return `${days}d ${hours}h`;
  if (hours > 0) return `${hours}h ${minutes}m`;
  return `${minutes}m`;
};
```

### 2. Chart Components

```typescript
// src/components/charts/LineChart.tsx
import React from 'react';
import { LineChart as RechartsLineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

interface LineChartProps {
  data: Array<Record<string, any>>;
  xField: string;
  yField: string;
  height?: number;
  color?: string;
  showGrid?: boolean;
  showTooltip?: boolean;
}

export const LineChart: React.FC<LineChartProps> = ({
  data,
  xField,
  yField,
  height = 300,
  color = '#3B82F6',
  showGrid = true,
  showTooltip = true
}) => {
  const formatXAxis = (value: any) => {
    if (xField.includes('time') || xField.includes('date')) {
      return new Date(value).toLocaleTimeString('en-US', { 
        hour12: false, 
        hour: '2-digit', 
        minute: '2-digit' 
      });
    }
    return value;
  };

  const formatTooltip = (value: any, name: string) => {
    if (typeof value === 'number') {
      return [value.toLocaleString(), name];
    }
    return [value, name];
  };

  return (
    <div style={{ width: '100%', height }}>
      <ResponsiveContainer>
        <RechartsLineChart data={data}>
          {showGrid && <CartesianGrid strokeDasharray="3 3" />}
          <XAxis 
            dataKey={xField} 
            tickFormatter={formatXAxis}
            stroke="#6B7280"
          />
          <YAxis stroke="#6B7280" />
          {showTooltip && (
            <Tooltip 
              formatter={formatTooltip}
              labelFormatter={formatXAxis}
              contentStyle={{
                backgroundColor: '#F9FAFB',
                border: '1px solid #E5E7EB',
                borderRadius: '6px'
              }}
            />
          )}
          <Line 
            type="monotone" 
            dataKey={yField} 
            stroke={color} 
            strokeWidth={2}
            dot={false}
            activeDot={{ r: 4, fill: color }}
          />
        </RechartsLineChart>
      </ResponsiveContainer>
    </div>
  );
};
```

## State Management

### 1. Custom Hooks

```typescript
// src/hooks/useSystemStatus.ts
import { useQuery } from '@tanstack/react-query';
import { systemService } from '@/services/systemService';
import { SystemStatus } from '@/types/system';

interface UseSystemStatusOptions {
  refetchInterval?: number;
  enabled?: boolean;
}

export const useSystemStatus = (options: UseSystemStatusOptions = {}) => {
  const { refetchInterval = 30000, enabled = true } = options;

  return useQuery<SystemStatus>({
    queryKey: ['system', 'status'],
    queryFn: () => systemService.getStatus(),
    refetchInterval,
    enabled,
    staleTime: 25000, // Consider data stale after 25 seconds
    cacheTime: 60000, // Keep in cache for 1 minute
    retry: 3,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
    onError: (error) => {
      console.error('Failed to fetch system status:', error);
    }
  });
};
```

```typescript
// src/hooks/useWebSocket.ts
import { useEffect, useRef, useState } from 'react';
import { useQueryClient } from '@tanstack/react-query';

interface UseWebSocketOptions {
  url: string;
  enabled?: boolean;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
}

export const useWebSocket = (options: UseWebSocketOptions) => {
  const { url, enabled = true, reconnectInterval = 5000, maxReconnectAttempts = 5 } = options;
  const [connectionStatus, setConnectionStatus] = useState<'connecting' | 'connected' | 'disconnected'>('disconnected');
  const [error, setError] = useState<string | null>(null);
  
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectAttemptsRef = useRef(0);
  const queryClient = useQueryClient();

  const connect = () => {
    if (!enabled) return;

    try {
      setConnectionStatus('connecting');
      wsRef.current = new WebSocket(url);

      wsRef.current.onopen = () => {
        setConnectionStatus('connected');
        setError(null);
        reconnectAttemptsRef.current = 0;
        console.log('WebSocket connected');
      };

      wsRef.current.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data);
          handleMessage(data);
        } catch (error) {
          console.error('Failed to parse WebSocket message:', error);
        }
      };

      wsRef.current.onclose = () => {
        setConnectionStatus('disconnected');
        console.log('WebSocket disconnected');
        
        // Attempt to reconnect
        if (reconnectAttemptsRef.current < maxReconnectAttempts) {
          reconnectAttemptsRef.current++;
          setTimeout(connect, reconnectInterval);
        }
      };

      wsRef.current.onerror = (error) => {
        setError('WebSocket connection error');
        console.error('WebSocket error:', error);
      };
    } catch (error) {
      setError('Failed to create WebSocket connection');
      setConnectionStatus('disconnected');
    }
  };

  const handleMessage = (data: any) => {
    // Update relevant queries based on message type
    switch (data.type) {
      case 'system_status':
        queryClient.setQueryData(['system', 'status'], data.payload);
        break;
      case 'collection_update':
        queryClient.invalidateQueries({ queryKey: ['system', 'collections'] });
        break;
      case 'market_update':
        queryClient.invalidateQueries({ queryKey: ['market'] });
        break;
      default:
        console.log('Unknown message type:', data.type);
    }
  };

  const disconnect = () => {
    if (wsRef.current) {
      wsRef.current.close();
      wsRef.current = null;
    }
  };

  useEffect(() => {
    if (enabled) {
      connect();
    }

    return () => {
      disconnect();
    };
  }, [enabled, url]);

  return {
    connectionStatus,
    error,
    connect,
    disconnect
  };
};
```

### 2. Global State with Zustand

```typescript
// src/store/appStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AppState {
  // Theme management
  theme: 'light' | 'dark';
  setTheme: (theme: 'light' | 'dark') => void;

  // Dashboard settings
  refreshInterval: number;
  setRefreshInterval: (interval: number) => void;

  // User preferences
  notifications: boolean;
  setNotifications: (enabled: boolean) => void;

  // Selected timeframe
  selectedTimeframe: 'minute' | 'hour' | 'day' | 'week' | 'month';
  setSelectedTimeframe: (timeframe: 'minute' | 'hour' | 'day' | 'week' | 'month') => void;

  // Alert settings
  alertThresholds: {
    apiUsageWarning: number;
    missingDataThreshold: number;
    systemHealthTimeout: number;
  };
  setAlertThresholds: (thresholds: Partial<AppState['alertThresholds']>) => void;
}

export const useAppStore = create<AppState>()(
  persist(
    (set, get) => ({
      // Initial state
      theme: 'light',
      refreshInterval: 30000,
      notifications: true,
      selectedTimeframe: 'hour',
      alertThresholds: {
        apiUsageWarning: 80,
        missingDataThreshold: 5,
        systemHealthTimeout: 10000
      },

      // Actions
      setTheme: (theme) => set({ theme }),
      setRefreshInterval: (refreshInterval) => set({ refreshInterval }),
      setNotifications: (notifications) => set({ notifications }),
      setSelectedTimeframe: (selectedTimeframe) => set({ selectedTimeframe }),
      setAlertThresholds: (thresholds) => 
        set((state) => ({
          alertThresholds: { ...state.alertThresholds, ...thresholds }
        }))
    }),
    {
      name: 'pulseguard-settings',
      partialize: (state) => ({
        theme: state.theme,
        refreshInterval: state.refreshInterval,
        notifications: state.notifications,
        alertThresholds: state.alertThresholds
      })
    }
  )
);
```

## API Integration

### 1. Service Layer

```typescript
// src/services/systemService.ts
import { apiClient } from './api';
import { SystemStatus, CollectionHistory, DatabaseHealth } from '@/types/system';

export class SystemService {
  async getStatus(): Promise<SystemStatus> {
    const response = await apiClient.get<SystemStatus>('/system/status');
    return response.data;
  }

  async getCollectionHistory(params?: {
    limit?: number;
    offset?: number;
    status?: string;
  }): Promise<CollectionHistory[]> {
    const response = await apiClient.get<CollectionHistory[]>('/system/collections', {
      params
    });
    return response.data;
  }

  async getDatabaseHealth(): Promise<DatabaseHealth> {
    const response = await apiClient.get<DatabaseHealth>('/system/database');
    return response.data;
  }

  async getDataCompleteness(timeframe?: string): Promise<any> {
    try {
      const response = await apiClient.get('/system/completeness', {
        params: timeframe ? { timeframe } : undefined
      });
      return response.data;
    } catch (error: any) {
      // Fallback for endpoints not yet implemented
      if (error.response?.status === 404) {
        return this.getFallbackCompleteness();
      }
      throw error;
    }
  }

  private async getFallbackCompleteness(): Promise<any> {
    // Generate mock completeness data based on collection history
    const collections = await this.getCollectionHistory({ limit: 10 });
    return {
      overall_completeness: 95.5,
      missing_coins: collections.filter(c => c.status === 'failed').length,
      last_update: new Date().toISOString(),
      timeframe_stats: {
        minute: { completeness: 98.2, missing_count: 12 },
        hour: { completeness: 99.1, missing_count: 5 },
        day: { completeness: 99.8, missing_count: 1 }
      }
    };
  }
}

export const systemService = new SystemService();
```

### 2. Error Handling

```typescript
// src/services/api.ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import { toast } from 'react-hot-toast';

class ApiClient {
  private client: AxiosInstance;
  private baseURL: string;
  private apiKey: string;

  constructor() {
    this.baseURL = import.meta.env.VITE_API_BASE_URL;
    this.apiKey = import.meta.env.VITE_API_KEY;

    this.client = axios.create({
      baseURL: this.baseURL,
      timeout: 10000,
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey
      }
    });

    this.setupInterceptors();
  }

  private setupInterceptors() {
    // Request interceptor
    this.client.interceptors.request.use(
      (config) => {
        // Add timestamp to prevent caching
        if (config.method === 'get') {
          config.params = {
            ...config.params,
            _t: Date.now()
          };
        }
        return config;
      },
      (error) => {
        return Promise.reject(error);
      }
    );

    // Response interceptor
    this.client.interceptors.response.use(
      (response) => {
        return response;
      },
      (error: AxiosError) => {
        return this.handleError(error);
      }
    );
  }

  private handleError(error: AxiosError): Promise<never> {
    const status = error.response?.status;
    const message = error.response?.data?.message || error.message;

    switch (status) {
      case 401:
        toast.error('Invalid API key. Please check your configuration.');
        break;
      case 403:
        toast.error('Access denied. Insufficient permissions.');
        break;
      case 404:
        // Don't show toast for 404s as they might be expected (fallback scenarios)
        break;
      case 429:
        toast.error('Rate limit exceeded. Please try again later.');
        break;
      case 500:
      case 502:
      case 503:
        toast.error('Server error. Please try again later.');
        break;
      default:
        if (error.code === 'NETWORK_ERROR' || error.code === 'ECONNABORTED') {
          toast.error('Network error. Please check your connection.');
        } else {
          toast.error(`Request failed: ${message}`);
        }
    }

    return Promise.reject(error);
  }

  // HTTP methods
  async get<T>(url: string, config?: any): Promise<{ data: T }> {
    const response = await this.client.get(url, config);
    return { data: response.data.data || response.data };
  }

  async post<T>(url: string, data?: any, config?: any): Promise<{ data: T }> {
    const response = await this.client.post(url, data, config);
    return { data: response.data.data || response.data };
  }

  async put<T>(url: string, data?: any, config?: any): Promise<{ data: T }> {
    const response = await this.client.put(url, data, config);
    return { data: response.data.data || response.data };
  }

  async delete<T>(url: string, config?: any): Promise<{ data: T }> {
    const response = await this.client.delete(url, config);
    return { data: response.data.data || response.data };
  }
}

export const apiClient = new ApiClient();
```

## Testing

### 1. Component Testing

```typescript
// src/components/__tests__/SystemMetrics.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { SystemMetrics } from '../monitoring/SystemMetrics';
import { systemService } from '@/services/systemService';

// Mock the service
jest.mock('@/services/systemService');
const mockSystemService = systemService as jest.Mocked<typeof systemService>;

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  });

  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('SystemMetrics', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('displays system status correctly', async () => {
    const mockData = {
      status: 'healthy',
      uptime: 3600,
      collectionsToday: 1440,
      successRate: 99.5,
      apiUsage: { callsToday: 2500 }
    };

    mockSystemService.getStatus.mockResolvedValue(mockData);

    render(<SystemMetrics />, { wrapper: createWrapper() });

    await waitFor(() => {
      expect(screen.getByText('HEALTHY')).toBeInTheDocument();
      expect(screen.getByText('1h')).toBeInTheDocument();
      expect(screen.getByText('1440')).toBeInTheDocument();
      expect(screen.getByText('99.5%')).toBeInTheDocument();
      expect(screen.getByText('2500')).toBeInTheDocument();
    });
  });

  test('handles loading state', () => {
    mockSystemService.getStatus.mockImplementation(
      () => new Promise(resolve => setTimeout(resolve, 1000))
    );

    render(<SystemMetrics />, { wrapper: createWrapper() });

    expect(screen.getByText(/loading/i)).toBeInTheDocument();
  });

  test('handles error state', async () => {
    mockSystemService.getStatus.mockRejectedValue(
      new Error('API unavailable')
    );

    render(<SystemMetrics />, { wrapper: createWrapper() });

    await waitFor(() => {
      expect(screen.getByText('Error Loading System Metrics')).toBeInTheDocument();
    });
  });
});
```

### 2. Hook Testing

```typescript
// src/hooks/__tests__/useSystemStatus.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useSystemStatus } from '../useSystemStatus';
import { systemService } from '@/services/systemService';

jest.mock('@/services/systemService');
const mockSystemService = systemService as jest.Mocked<typeof systemService>;

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  });

  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('useSystemStatus', () => {
  test('fetches system status successfully', async () => {
    const mockData = {
      status: 'healthy',
      uptime: 3600,
      collectionsToday: 1440
    };

    mockSystemService.getStatus.mockResolvedValue(mockData);

    const { result } = renderHook(() => useSystemStatus(), {
      wrapper: createWrapper()
    });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual(mockData);
  });

  test('handles error correctly', async () => {
    mockSystemService.getStatus.mockRejectedValue(
      new Error('Network error')
    );

    const { result } = renderHook(() => useSystemStatus(), {
      wrapper: createWrapper()
    });

    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    expect(result.current.error).toBeInstanceOf(Error);
  });
});
```

## Performance Optimization

### 1. Component Memoization

```typescript
// Memoize expensive components
const SystemMetrics = React.memo(({ refreshInterval, showCharts }: SystemMetricsProps) => {
  const { data } = useSystemStatus({ refetchInterval: refreshInterval });
  
  const chartData = useMemo(() => 
    processChartData(data?.metrics), [data?.metrics]
  );

  return (
    <Card>
      {/* Component content */}
      {showCharts && <LineChart data={chartData} />}
    </Card>
  );
});
```

### 2. Query Optimization

```typescript
// src/hooks/useOptimizedQueries.ts
export const useOptimizedSystemData = () => {
  const queryClient = useQueryClient();

  // Prefetch related data
  const prefetchCollectionHistory = useCallback(() => {
    queryClient.prefetchQuery({
      queryKey: ['system', 'collections'],
      queryFn: () => systemService.getCollectionHistory({ limit: 5 }),
      staleTime: 60000
    });
  }, [queryClient]);

  // Use query with optimistic updates
  const systemStatusQuery = useQuery({
    queryKey: ['system', 'status'],
    queryFn: systemService.getStatus,
    refetchInterval: 30000,
    staleTime: 25000,
    select: useCallback((data: SystemStatus) => ({
      ...data,
      // Computed values
      uptimeFormatted: formatUptime(data.uptime),
      isHealthy: data.status === 'healthy',
      needsAttention: data.successRate < 95
    }), [])
  });

  useEffect(() => {
    if (systemStatusQuery.data?.isHealthy) {
      prefetchCollectionHistory();
    }
  }, [systemStatusQuery.data?.isHealthy, prefetchCollectionHistory]);

  return systemStatusQuery;
};
```

## Best Practices

1. **Component Organization**: Keep components small and focused on single responsibilities
2. **Type Safety**: Use TypeScript interfaces for all props and API responses
3. **Error Boundaries**: Implement error boundaries for graceful error handling
4. **Performance**: Use React.memo, useMemo, and useCallback appropriately
5. **Testing**: Write unit tests for components and integration tests for hooks
6. **Accessibility**: Ensure components are accessible with proper ARIA labels
7. **Code Splitting**: Use lazy loading for routes and heavy components
8. **State Management**: Use React Query for server state and Zustand for client state
9. **Styling**: Use Tailwind CSS classes consistently and create reusable component variants
10. **Documentation**: Document complex components and custom hooks with JSDoc comments

This development guide provides a comprehensive foundation for building and maintaining the PulseGuard dashboard with modern React patterns and best practices.
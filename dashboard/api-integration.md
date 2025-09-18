# PulseGuard Dashboard API Integration

## Overview

This document outlines how the PulseGuard Dashboard integrates with the PulseGuard API service to provide real-time monitoring and data visualization capabilities.

## API Client Architecture

### Base API Client

```typescript
// src/services/api.ts
export class ApiClient {
  private baseURL: string;
  private apiKey: string;
  private timeout = 10000;

  constructor(config: ApiConfig) {
    this.baseURL = config.baseURL;
    this.apiKey = config.apiKey;
  }

  async request<T>(endpoint: string, options?: RequestOptions): Promise<ApiResponse<T>> {
    const url = `${this.baseURL}${endpoint}`;
    const config: RequestInit = {
      method: options?.method || 'GET',
      headers: {
        'X-API-Key': this.apiKey,
        'Content-Type': 'application/json',
        ...options?.headers
      },
      signal: AbortSignal.timeout(this.timeout),
      ...options
    };

    try {
      const response = await fetch(url, config);
      return this.handleResponse<T>(response);
    } catch (error) {
      throw this.handleError(error);
    }
  }

  private async handleResponse<T>(response: Response): Promise<ApiResponse<T>> {
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      throw new ApiError(response.status, errorData.message || response.statusText);
    }
    return response.json();
  }
}
```

### Service Layer

#### System Service
```typescript
// src/services/systemService.ts
export class SystemService {
  constructor(private apiClient: ApiClient) {}

  async getSystemStatus(): Promise<SystemStatus> {
    const response = await this.apiClient.request<SystemStatus>('/system/status');
    return response.data;
  }

  async getCollectionHistory(params?: CollectionHistoryParams): Promise<CollectionHistory[]> {
    const queryString = params ? new URLSearchParams(params).toString() : '';
    const endpoint = `/system/collections${queryString ? `?${queryString}` : ''}`;
    const response = await this.apiClient.request<CollectionHistory[]>(endpoint);
    return response.data;
  }

  async getDatabaseHealth(): Promise<DatabaseHealth> {
    const response = await this.apiClient.request<DatabaseHealth>('/system/database');
    return response.data;
  }

  async getDataCompleteness(timeframe?: string): Promise<DataCompleteness> {
    const endpoint = `/system/completeness${timeframe ? `?timeframe=${timeframe}` : ''}`;
    try {
      const response = await this.apiClient.request<DataCompleteness>(endpoint);
      return response.data;
    } catch (error) {
      // Fallback for new endpoints not yet implemented
      if (error instanceof ApiError && error.status === 404) {
        return this.getFallbackCompleteness();
      }
      throw error;
    }
  }

  private async getFallbackCompleteness(): Promise<DataCompleteness> {
    // Fallback implementation using existing endpoints
    const collections = await this.getCollectionHistory({ limit: 10 });
    return this.computeCompletenessFromCollections(collections);
  }
}
```

#### Market Service
```typescript
// src/services/marketService.ts
export class MarketService {
  constructor(private apiClient: ApiClient) {}

  async getTopCoins(count: number = 10): Promise<Coin[]> {
    const response = await this.apiClient.request<Coin[]>(`/market/top/${count}`);
    return response.data;
  }

  async getMarketSummary(): Promise<MarketSummary> {
    const response = await this.apiClient.request<MarketSummary>('/market/summary');
    return response.data;
  }

  async getCoinHistory(coinId: string, params: HistoryParams): Promise<PriceHistory[]> {
    const queryString = new URLSearchParams(params).toString();
    const endpoint = `/coins/${coinId}/history?${queryString}`;
    const response = await this.apiClient.request<PriceHistory[]>(endpoint);
    return response.data;
  }

  async getMultiCoinHistory(params: MultiCoinHistoryParams): Promise<MultiCoinHistory> {
    const queryString = new URLSearchParams(params).toString();
    const endpoint = `/market/history?${queryString}`;
    const response = await this.apiClient.request<MultiCoinHistory>(endpoint);
    return response.data;
  }
}
```

## React Query Integration

### Query Key Factory

```typescript
// src/services/queryKeys.ts
export const queryKeys = {
  // System queries
  system: {
    all: ['system'] as const,
    status: () => [...queryKeys.system.all, 'status'] as const,
    collections: (filters?: CollectionFilters) => 
      [...queryKeys.system.all, 'collections', filters] as const,
    database: () => [...queryKeys.system.all, 'database'] as const,
    completeness: (timeframe?: string) => 
      [...queryKeys.system.all, 'completeness', timeframe] as const
  },
  
  // Market queries
  market: {
    all: ['market'] as const,
    summary: () => [...queryKeys.market.all, 'summary'] as const,
    top: (count: number) => [...queryKeys.market.all, 'top', count] as const,
    history: (params: HistoryParams) => 
      [...queryKeys.market.all, 'history', params] as const
  },
  
  // Coin queries
  coins: {
    all: ['coins'] as const,
    detail: (id: string) => [...queryKeys.coins.all, 'detail', id] as const,
    history: (id: string, params: HistoryParams) => 
      [...queryKeys.coins.all, 'history', id, params] as const
  }
} as const;
```

### Custom Hooks

#### System Status Hook
```typescript
// src/hooks/useSystemStatus.ts
export const useSystemStatus = (options?: UseQueryOptions<SystemStatus>) => {
  return useQuery({
    queryKey: queryKeys.system.status(),
    queryFn: () => systemService.getSystemStatus(),
    refetchInterval: 30000, // Refresh every 30 seconds
    staleTime: 25000, // Consider stale after 25 seconds
    retry: 3,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
    ...options
  });
};

export const useSystemStatusWithWebSocket = () => {
  const query = useSystemStatus();
  const queryClient = useQueryClient();

  useEffect(() => {
    const ws = new WebSocket(config.wsUrl);
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (data.type === 'system_status') {
        queryClient.setQueryData(queryKeys.system.status(), data.payload);
      }
    };

    return () => ws.close();
  }, [queryClient]);

  return query;
};
```

#### Data Completeness Hook
```typescript
// src/hooks/useDataCompleteness.ts
export const useDataCompleteness = (timeframe?: string) => {
  return useQuery({
    queryKey: queryKeys.system.completeness(timeframe),
    queryFn: () => systemService.getDataCompleteness(timeframe),
    refetchInterval: 60000, // Refresh every minute
    staleTime: 50000,
    select: (data) => ({
      ...data,
      completenessPercentage: calculateCompleteness(data),
      missingCoins: identifyMissingCoins(data),
      alerts: generateCompletenessAlerts(data)
    })
  });
};
```

#### Market Data Hook
```typescript
// src/hooks/useMarketData.ts
export const useTopCoins = (count: number = 10) => {
  return useQuery({
    queryKey: queryKeys.market.top(count),
    queryFn: () => marketService.getTopCoins(count),
    staleTime: 300000, // 5 minutes
    cacheTime: 600000  // 10 minutes
  });
};

export const useCoinHistory = (coinId: string, params: HistoryParams) => {
  return useQuery({
    queryKey: queryKeys.coins.history(coinId, params),
    queryFn: () => marketService.getCoinHistory(coinId, params),
    enabled: !!coinId,
    staleTime: getStaleTimeForTimeframe(params.timeframe),
    select: (data) => ({
      data,
      processedData: processChartData(data),
      statistics: calculateStatistics(data)
    })
  });
};

const getStaleTimeForTimeframe = (timeframe: string): number => {
  const staleTimeMap = {
    minute: 30000,   // 30 seconds
    hour: 300000,    // 5 minutes
    day: 1800000,    // 30 minutes
    week: 3600000,   // 1 hour
    month: 7200000   // 2 hours
  };
  return staleTimeMap[timeframe] || 300000;
};
```

## Error Handling

### Error Types

```typescript
// src/types/errors.ts
export class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
    public details?: unknown
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export class NetworkError extends Error {
  constructor(message: string, public originalError: Error) {
    super(message);
    this.name = 'NetworkError';
  }
}

export class ValidationError extends Error {
  constructor(message: string, public field: string) {
    super(message);
    this.name = 'ValidationError';
  }
}
```

### Global Error Boundary

```typescript
// src/components/ErrorBoundary.tsx
export const ErrorBoundary: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundaryComponent
          onError={(error, errorInfo) => {
            console.error('Error caught by boundary:', error, errorInfo);
            // Log to monitoring service
            logError(error, errorInfo);
          }}
          fallbackRender={({ error, resetErrorBoundary }) => (
            <ErrorFallback 
              error={error} 
              resetErrorBoundary={() => {
                reset();
                resetErrorBoundary();
              }} 
            />
          )}
        >
          {children}
        </ErrorBoundaryComponent>
      )}
    </QueryErrorResetBoundary>
  );
};
```

### Error Recovery

```typescript
// src/hooks/useErrorRecovery.ts
export const useErrorRecovery = () => {
  const queryClient = useQueryClient();
  
  const retryFailedQueries = useCallback(() => {
    queryClient.refetchQueries({
      predicate: (query) => query.state.status === 'error'
    });
  }, [queryClient]);

  const clearErrorState = useCallback(() => {
    queryClient.resetQueries({
      predicate: (query) => query.state.status === 'error'
    });
  }, [queryClient]);

  return { retryFailedQueries, clearErrorState };
};
```

## Real-time Updates

### WebSocket Integration

```typescript
// src/services/websocketService.ts
export class WebSocketService {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectInterval = 1000;

  constructor(
    private url: string,
    private onMessage: (data: any) => void,
    private onError?: (error: Event) => void
  ) {}

  connect(): void {
    try {
      this.ws = new WebSocket(this.url);
      
      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.reconnectAttempts = 0;
      };

      this.ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        this.onMessage(data);
      };

      this.ws.onclose = (event) => {
        console.log('WebSocket disconnected:', event.reason);
        this.attemptReconnect();
      };

      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        this.onError?.(error);
      };
    } catch (error) {
      console.error('Failed to create WebSocket connection:', error);
      this.attemptReconnect();
    }
  }

  private attemptReconnect(): void {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = this.reconnectInterval * Math.pow(2, this.reconnectAttempts - 1);
      
      setTimeout(() => {
        console.log(`Attempting to reconnect (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
        this.connect();
      }, delay);
    }
  }

  disconnect(): void {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}
```

### Real-time Data Hook

```typescript
// src/hooks/useRealtimeData.ts
export const useRealtimeData = () => {
  const queryClient = useQueryClient();
  const wsRef = useRef<WebSocketService | null>(null);

  useEffect(() => {
    const handleWebSocketMessage = (data: RealtimeMessage) => {
      switch (data.type) {
        case 'system_status':
          queryClient.setQueryData(queryKeys.system.status(), data.payload);
          break;
        
        case 'collection_update':
          queryClient.invalidateQueries({ queryKey: queryKeys.system.collections() });
          break;
        
        case 'market_update':
          queryClient.setQueryData(queryKeys.market.summary(), data.payload);
          break;
        
        case 'price_update':
          // Update specific coin data
          const { coinId, priceData } = data.payload;
          queryClient.setQueryData(
            queryKeys.coins.detail(coinId),
            (oldData: any) => ({ ...oldData, ...priceData })
          );
          break;
      }
    };

    wsRef.current = new WebSocketService(
      config.wsUrl,
      handleWebSocketMessage,
      (error) => console.error('WebSocket error:', error)
    );

    wsRef.current.connect();

    return () => {
      wsRef.current?.disconnect();
    };
  }, [queryClient]);

  return {
    isConnected: wsRef.current !== null,
    disconnect: () => wsRef.current?.disconnect()
  };
};
```

## Performance Optimizations

### Request Batching

```typescript
// src/services/batchService.ts
export class BatchService {
  private batchQueue: Map<string, BatchRequest[]> = new Map();
  private batchTimeout: NodeJS.Timeout | null = null;

  addToBatch(endpoint: string, params: any): Promise<any> {
    return new Promise((resolve, reject) => {
      const batchKey = this.getBatchKey(endpoint);
      
      if (!this.batchQueue.has(batchKey)) {
        this.batchQueue.set(batchKey, []);
      }
      
      this.batchQueue.get(batchKey)!.push({
        params,
        resolve,
        reject
      });

      this.scheduleBatchExecution();
    });
  }

  private scheduleBatchExecution(): void {
    if (this.batchTimeout) return;
    
    this.batchTimeout = setTimeout(() => {
      this.executeBatches();
      this.batchTimeout = null;
    }, 100); // 100ms batch window
  }

  private async executeBatches(): Promise<void> {
    for (const [endpoint, requests] of this.batchQueue.entries()) {
      try {
        const batchResponse = await this.executeBatchRequest(endpoint, requests);
        this.resolveBatchRequests(requests, batchResponse);
      } catch (error) {
        this.rejectBatchRequests(requests, error);
      }
    }
    
    this.batchQueue.clear();
  }
}
```

### Response Caching

```typescript
// src/services/cacheService.ts
export class CacheService {
  private cache = new Map<string, CacheEntry>();
  private maxSize = 1000;
  private defaultTTL = 300000; // 5 minutes

  get<T>(key: string): T | null {
    const entry = this.cache.get(key);
    
    if (!entry) return null;
    
    if (Date.now() > entry.expiry) {
      this.cache.delete(key);
      return null;
    }
    
    return entry.data as T;
  }

  set<T>(key: string, data: T, ttl = this.defaultTTL): void {
    if (this.cache.size >= this.maxSize) {
      this.evictOldest();
    }
    
    this.cache.set(key, {
      data,
      expiry: Date.now() + ttl,
      lastAccessed: Date.now()
    });
  }

  private evictOldest(): void {
    let oldestKey = '';
    let oldestTime = Date.now();
    
    for (const [key, entry] of this.cache.entries()) {
      if (entry.lastAccessed < oldestTime) {
        oldestTime = entry.lastAccessed;
        oldestKey = key;
      }
    }
    
    if (oldestKey) {
      this.cache.delete(oldestKey);
    }
  }
}
```

## Testing Integration

### API Mock Service

```typescript
// src/testing/mockApiService.ts
export const createMockApiService = () => {
  const mockData = {
    systemStatus: {
      status: 'healthy',
      uptime: 3600,
      lastCollection: new Date().toISOString(),
      collectionsToday: 1440,
      successRate: 99.5
    },
    
    marketSummary: {
      totalCoins: 1000,
      totalMarketCap: 2500000000000,
      totalVolume: 95000000000,
      topGainer: { id: 'bitcoin', change: 5.2 },
      topLoser: { id: 'ethereum', change: -2.1 }
    }
  };

  return {
    system: {
      getStatus: jest.fn().mockResolvedValue(mockData.systemStatus),
      getCollections: jest.fn().mockResolvedValue([]),
      getDatabaseHealth: jest.fn().mockResolvedValue({ status: 'healthy' })
    },
    market: {
      getSummary: jest.fn().mockResolvedValue(mockData.marketSummary),
      getTopCoins: jest.fn().mockResolvedValue([])
    }
  };
};
```

### Integration Test Setup

```typescript
// src/testing/testSetup.ts
export const setupApiTests = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  });

  const TestWrapper: React.FC<{ children: React.ReactNode }> = ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );

  return { queryClient, TestWrapper };
};
```

---

*Last updated: 2025-09-18*
*Version: 1.0*
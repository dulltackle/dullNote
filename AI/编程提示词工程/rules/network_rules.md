---
description: `此规则适用于Electron项目的网络客户端调用规范，提供HTTP客户端、IPC通信和WebSocket连接的标准实现，确保代码质量与可维护性。`
globs:
alwaysApply: false
---
# Electron项目网络客户端调用规范

## 关键规则

- **所有网络客户端必须放在src/services目录下**
- **必须遵循统一的命名规范：服务功能+Service命名方式**
- **所有异步操作必须使用async/await或Promise，支持错误处理和日志记录**
- **网络客户端必须从配置文件读取配置，不允许硬编码**
- **网络调用必须记录完整的请求和响应日志**
- **必须实现统一的错误处理和返回机制**
- **应对关键调用实现缓存机制，减少直接调用次数**
- **请求和响应必须使用TypeScript接口定义类型**
- **客户端调用需要进行合理的超时控制**
- **主进程和渲染进程间通信必须使用IPC机制**

## HTTP客户端标准实现

### 客户端定义规范

```typescript
// 客户端标准定义
interface HttpClientConfig {
  baseURL: string;
  timeout: number;
  headers?: Record<string, string>;
}

class XXXHttpService {
  private config: HttpClientConfig;
  private logger: Logger;

  constructor(config: HttpClientConfig, logger: Logger) {
    this.config = config;
    this.logger = logger;
  }

  // 必须定义统一的初始化方法
  static async create(): Promise<XXXHttpService> {
    const config = await ConfigService.getHttpConfig('XXXService');
    const logger = LoggerService.getLogger('XXXHttpService');
    return new XXXHttpService(config, logger);
  }
}
```

### 请求参数定义

```typescript
// 请求参数必须使用接口定义
interface RequestParams {
  field1: string;
  field2: number;
  // ...
}

// 响应结构定义
interface ResponseData<T = any> {
  code: number;
  message: string;
  data: T;
  success: boolean;
}
```

### 标准HTTP调用实现

```typescript
// 标准HTTP调用方法
async sendRequest<T>(params: RequestParams): Promise<ResponseData<T>> {
  try {
    // 1. 从配置获取URL配置
    const config = await ConfigService.getHttpConfig('XXXService');
    if (!config.baseURL) {
      throw new NetworkError('URL configuration missing', 'CONFIG_ERROR');
    }

    // 2. 构建请求参数
    const requestId = generateRequestId();
    const requestData = {
      ...params,
      requestId,
      timestamp: Date.now()
    };

    // 3. 构建请求选项
    const requestOptions: RequestInit = {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Request-ID': requestId,
        ...config.headers
      },
      body: JSON.stringify(requestData),
      signal: AbortSignal.timeout(config.timeout || 10000)
    };

    // 4. 发送请求并记录日志
    const url = `${config.baseURL}/api/endpoint`;
    this.logger.info('HTTP Request', { url, params: requestData });

    const response = await fetch(url, requestOptions);

    // 5. 错误处理
    if (!response.ok) {
      throw new NetworkError(
        `HTTP ${response.status}: ${response.statusText}`,
        'HTTP_ERROR',
        response.status
      );
    }

    // 6. 解析响应
    const responseData: ResponseData<T> = await response.json();

    // 7. 记录响应日志
    this.logger.info('HTTP Response', {
      requestId,
      status: response.status,
      data: responseData
    });

    // 8. 业务错误处理
    if (!responseData.success || responseData.code !== 200) {
      throw new BusinessError(
        responseData.message || 'Business logic error',
        'BUSINESS_ERROR',
        responseData.code
      );
    }

    return responseData;
  } catch (error) {
    this.logger.error('HTTP Request Failed', { params, error });
    throw error;
  }
}
```

### 带缓存的HTTP调用

```typescript
async getDataWithCache<T>(
  key: string,
  params: RequestParams,
  ttl: number = 5000
): Promise<ResponseData<T>> {
  const cacheKey = `http_cache_${key}_${JSON.stringify(params)}`;

  // 尝试从缓存获取
  const cached = await CacheService.get<ResponseData<T>>(cacheKey);
  if (cached) {
    this.logger.debug('Cache Hit', { cacheKey });
    return cached;
  }

  // 缓存未命中，发起请求
  try {
    const result = await this.sendRequest<T>(params);

    // 存储到缓存
    await CacheService.set(cacheKey, result, ttl);
    this.logger.debug('Cache Set', { cacheKey, ttl });

    return result;
  } catch (error) {
    this.logger.error('Request with cache failed', { key, params, error });
    throw error;
  }
}
```

## IPC通信标准实现

### 主进程服务定义

```typescript
// 主进程服务定义
class MainProcessService {
  private httpService: XXXHttpService;
  private logger: Logger;

  constructor() {
    this.logger = LoggerService.getLogger('MainProcessService');
    this.setupIpcHandlers();
  }

  private setupIpcHandlers(): void {
    // 注册IPC处理器
    ipcMain.handle('service:request', this.handleServiceRequest.bind(this));
    ipcMain.handle('service:config', this.handleConfigRequest.bind(this));
  }

  private async handleServiceRequest(
    event: IpcMainInvokeEvent,
    serviceName: string,
    method: string,
    params: any
  ): Promise<any> {
    const requestId = generateRequestId();

    try {
      this.logger.info('IPC Request', { requestId, serviceName, method, params });

      // 根据服务名称路由到对应的服务
      const service = await this.getService(serviceName);
      const result = await service[method](params);

      this.logger.info('IPC Response', { requestId, result });
      return { success: true, data: result };
    } catch (error) {
      this.logger.error('IPC Request Failed', { requestId, serviceName, method, error });
      return {
        success: false,
        error: {
          message: error.message,
          code: error.code || 'UNKNOWN_ERROR'
        }
      };
    }
  }
}
```

### 渲染进程客户端

```typescript
// 渲染进程客户端
class RendererServiceClient {
  private logger: Logger;

  constructor() {
    this.logger = LoggerService.getLogger('RendererServiceClient');
  }

  async callService<T>(
    serviceName: string,
    method: string,
    params: any
  ): Promise<T> {
    try {
      const result = await window.electronAPI.invoke(
        'service:request',
        serviceName,
        method,
        params
      );

      if (!result.success) {
        throw new ServiceError(
          result.error.message,
          result.error.code
        );
      }

      return result.data;
    } catch (error) {
      this.logger.error('Service call failed', { serviceName, method, params, error });
      throw error;
    }
  }
}
```

## WebSocket连接标准实现

### WebSocket客户端定义

```typescript
interface WebSocketConfig {
  url: string;
  reconnectInterval: number;
  maxReconnectAttempts: number;
  heartbeatInterval: number;
}

class WebSocketService extends EventEmitter {
  private ws: WebSocket | null = null;
  private config: WebSocketConfig;
  private logger: Logger;
  private reconnectAttempts = 0;
  private heartbeatTimer: NodeJS.Timeout | null = null;

  constructor(config: WebSocketConfig) {
    super();
    this.config = config;
    this.logger = LoggerService.getLogger('WebSocketService');
  }

  async connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      try {
        this.ws = new WebSocket(this.config.url);

        this.ws.onopen = () => {
          this.logger.info('WebSocket connected', { url: this.config.url });
          this.reconnectAttempts = 0;
          this.startHeartbeat();
          this.emit('connected');
          resolve();
        };

        this.ws.onmessage = (event) => {
          this.handleMessage(event.data);
        };

        this.ws.onclose = () => {
          this.logger.warn('WebSocket disconnected');
          this.stopHeartbeat();
          this.emit('disconnected');
          this.attemptReconnect();
        };

        this.ws.onerror = (error) => {
          this.logger.error('WebSocket error', { error });
          this.emit('error', error);
          reject(error);
        };
      } catch (error) {
        this.logger.error('WebSocket connection failed', { error });
        reject(error);
      }
    });
  }

  private handleMessage(data: string): void {
    try {
      const message = JSON.parse(data);
      this.logger.debug('WebSocket message received', { message });
      this.emit('message', message);
    } catch (error) {
      this.logger.error('Failed to parse WebSocket message', { data, error });
    }
  }

  send(data: any): void {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      const message = JSON.stringify(data);
      this.ws.send(message);
      this.logger.debug('WebSocket message sent', { data });
    } else {
      this.logger.warn('WebSocket not connected, message not sent', { data });
    }
  }
}
```

## 错误处理规范

```typescript
// 标准错误类定义
class NetworkError extends Error {
  public readonly code: string;
  public readonly statusCode?: number;

  constructor(message: string, code: string, statusCode?: number) {
    super(message);
    this.name = 'NetworkError';
    this.code = code;
    this.statusCode = statusCode;
  }
}

class BusinessError extends Error {
  public readonly code: string;
  public readonly businessCode: number;

  constructor(message: string, code: string, businessCode: number) {
    super(message);
    this.name = 'BusinessError';
    this.code = code;
    this.businessCode = businessCode;
  }
}

// 错误处理示例
try {
  const result = await httpService.sendRequest(params);
  return result;
} catch (error) {
  if (error instanceof NetworkError) {
    // 网络错误处理
    logger.error('Network error occurred', { error });
    throw new ServiceError('网络请求失败', 'NETWORK_ERROR');
  } else if (error instanceof BusinessError) {
    // 业务错误处理
    logger.warn('Business error occurred', { error });
    throw error;
  } else {
    // 未知错误
    logger.error('Unknown error occurred', { error });
    throw new ServiceError('未知错误', 'UNKNOWN_ERROR');
  }
}
```

## 日志记录规范

```typescript
// 日志服务定义
class LoggerService {
  private static loggers = new Map<string, Logger>();

  static getLogger(name: string): Logger {
    if (!this.loggers.has(name)) {
      this.loggers.set(name, new Logger(name));
    }
    return this.loggers.get(name)!;
  }
}

class Logger {
  constructor(private name: string) {}

  info(message: string, meta?: any): void {
    this.log('info', message, meta);
  }

  warn(message: string, meta?: any): void {
    this.log('warn', message, meta);
  }

  error(message: string, meta?: any): void {
    this.log('error', message, meta);
  }

  debug(message: string, meta?: any): void {
    this.log('debug', message, meta);
  }

  private log(level: string, message: string, meta?: any): void {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      logger: this.name,
      message,
      requestId: getCurrentRequestId(),
      ...meta
    };

    // 敏感信息脱敏
    const sanitized = this.sanitizeLogData(logEntry);

    console.log(JSON.stringify(sanitized));

    // 可选：写入文件或发送到日志服务
    this.writeToFile(sanitized);
  }

  private sanitizeLogData(data: any): any {
    // 脱敏处理：密码、token等敏感信息
    const sensitiveFields = ['password', 'token', 'secret', 'key'];
    const sanitized = JSON.parse(JSON.stringify(data));

    const maskSensitiveData = (obj: any) => {
      for (const key in obj) {
        if (sensitiveFields.some(field => key.toLowerCase().includes(field))) {
          obj[key] = '***masked***';
        } else if (typeof obj[key] === 'object' && obj[key] !== null) {
          maskSensitiveData(obj[key]);
        }
      }
    };

    maskSensitiveData(sanitized);
    return sanitized;
  }
}
```

## 配置管理规范

```typescript
// 配置服务
class ConfigService {
  private static config: any = null;

  static async loadConfig(): Promise<void> {
    try {
      const configPath = path.join(__dirname, '../config/app.json');
      const configData = await fs.readFile(configPath, 'utf-8');
      this.config = JSON.parse(configData);
    } catch (error) {
      throw new Error(`Failed to load configuration: ${error.message}`);
    }
  }

  static async getHttpConfig(serviceName: string): Promise<HttpClientConfig> {
    if (!this.config) {
      await this.loadConfig();
    }

    const serviceConfig = this.config.services?.[serviceName];
    if (!serviceConfig) {
      throw new Error(`Configuration for service '${serviceName}' not found`);
    }

    return {
      baseURL: serviceConfig.baseURL,
      timeout: serviceConfig.timeout || 10000,
      headers: serviceConfig.headers || {}
    };
  }
}
```

## 示例

### HTTP客户端调用示例
```typescript
// 搜索服务示例
interface SearchParams {
  query: string;
  latitude: number;
  longitude: number;
}

interface SearchItem {
  id: string;
  name: string;
  distance: number;
}

interface SearchResponse {
  items: SearchItem[];
  total: number;
}

class SearchService {
  private httpService: XXXHttpService;
  private logger: Logger;

  constructor() {
    this.logger = LoggerService.getLogger('SearchService');
  }

  static async create(): Promise<SearchService> {
    const service = new SearchService();
    service.httpService = await XXXHttpService.create();
    return service;
  }

  async search(params: SearchParams): Promise<SearchResponse> {
    try {
      const response = await this.httpService.sendRequest<SearchResponse>({
        action: 'search',
        ...params
      });

      return response.data;
    } catch (error) {
      this.logger.error('Search request failed', { params, error });
      throw new ServiceError('搜索请求失败', 'SEARCH_ERROR');
    }
  }

  async searchWithCache(
    params: SearchParams,
    ttl: number = 30000
  ): Promise<SearchResponse> {
    const cacheKey = `search_${params.query}_${params.latitude}_${params.longitude}`;

    try {
      const response = await this.httpService.getDataWithCache<SearchResponse>(
        cacheKey,
        { action: 'search', ...params },
        ttl
      );

      return response.data;
    } catch (error) {
      this.logger.error('Cached search request failed', { params, error });
      throw error;
    }
  }
}
```

### 错误示例
```typescript
// 错误示例：硬编码URL和缺少日志记录
class BadSearchService {
  // 错误1: 硬编码URL
  // 错误2: 没有使用配置管理
  // 错误3: 没有错误处理
  async badSearch(query: string): Promise<any> {
    // 硬编码URL
    const url = 'http://search.example.com/api?query=' + query;

    // 没有超时控制
    const response = await fetch(url);

    // 没有日志记录
    // 没有错误处理
    return response.json();
  }
}

// 错误示例：IPC调用不规范
class BadIpcService {
  // 错误1: 不遵循标准接口定义
  // 错误2: 没有错误处理
  // 错误3: 没有日志记录
  async badIpcCall(data: any): Promise<any> {
    // 直接调用，没有错误处理
    return window.electronAPI.invoke('some-channel', data);
  }
}
```
# 建立WebSocket连接：在SourceFunction中初始化

## 核心概念

在 SourceFunction 的 `run()` 方法中建立 WebSocket 连接，这是实现币安数据源的第一步。

### 执行时机

WebSocket 连接应该在 `run()` 方法中建立，而不是构造函数中，因为：
1. **生命周期管理**：连接的生命周期与 `run()` 方法一致
2. **异常处理**：可以在 `run()` 中处理连接异常
3. **资源清理**：在 `cancel()` 中关闭连接

## 最小可用例子

### 使用 OkHttp

```java
public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private OkHttpClient httpClient;
    private WebSocket webSocket;

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        // 1. 创建HTTP客户端（在run()方法中）
        httpClient = new OkHttpClient.Builder()
            .pingInterval(20, TimeUnit.SECONDS)  // 保持连接活跃
            .readTimeout(0, TimeUnit.SECONDS)    // 不设置读取超时
            .build();

        // 2. 构建WebSocket请求
        Request request = new Request.Builder()
            .url("wss://stream.binance.com:9443/ws/btcusdt@trade")
            .build();

        // 3. 建立WebSocket连接
        webSocket = httpClient.newWebSocket(request, new WebSocketListener() {
            @Override
            public void onOpen(WebSocket webSocket, Response response) {
                logger.info("WebSocket connected to Binance");
            }

            @Override
            public void onMessage(WebSocket webSocket, String text) {
                try {
                    Trade trade = parseJson(text);
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(trade, trade.getTradeTime());
                    }
                } catch (Exception e) {
                    logger.error("Failed to process message", e);
                }
            }

            @Override
            public void onFailure(WebSocket webSocket, Throwable t, Response response) {
                logger.error("WebSocket connection failed", t);
                // 这里可以触发重连逻辑
            }

            @Override
            public void onClosed(WebSocket webSocket, int code, String reason) {
                logger.info("WebSocket closed: " + reason);
            }
        });

        // 4. 保持run()方法运行
        while (isRunning) {
            Thread.sleep(100);
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
        if (webSocket != null) {
            webSocket.close(1000, "Normal closure");
        }
        if (httpClient != null) {
            httpClient.dispatcher().executorService().shutdown();
        }
    }
}
```

### 使用 Java-WebSocket

```java
public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        // 1. 创建WebSocket客户端（在run()方法中）
        client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
            @Override
            public void onOpen(ServerHandshake handshake) {
                logger.info("WebSocket connected");
            }

            @Override
            public void onMessage(String message) {
                try {
                    Trade trade = parseJson(message);
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(trade, trade.getTradeTime());
                    }
                } catch (Exception e) {
                    logger.error("Failed to process message", e);
                }
            }

            @Override
            public void onClose(int code, String reason, boolean remote) {
                logger.info("WebSocket closed: " + reason);
            }

            @Override
            public void onError(Exception ex) {
                logger.error("WebSocket error", ex);
            }
        };

        // 2. 建立连接
        client.connect();

        // 3. 等待连接建立
        while (!client.isOpen() && isRunning) {
            Thread.sleep(100);
        }

        // 4. 保持run()方法运行
        while (isRunning) {
            Thread.sleep(100);
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
        if (client != null) {
            client.close();
        }
    }
}
```

## 关键要点

1. **在 `run()` 方法中建立连接**：不要构造函数中建立
2. **等待连接建立**：确保连接成功后再处理数据
3. **异常处理**：处理连接失败的情况
4. **资源清理**：在 `cancel()` 中关闭连接

## 连接配置

### OkHttp 配置

```java
OkHttpClient client = new OkHttpClient.Builder()
    .pingInterval(20, TimeUnit.SECONDS)  // 每20秒发送ping保持连接
    .readTimeout(0, TimeUnit.SECONDS)     // 不设置读取超时（流式数据）
    .connectTimeout(10, TimeUnit.SECONDS) // 连接超时10秒
    .build();
```

### Java-WebSocket 配置

```java
WebSocketClient client = new WebSocketClient(uri) {
    // 实现回调方法
};

// 设置连接超时
client.setConnectionLostTimeout(60);  // 60秒无响应认为连接丢失
```

## 什么时候你需要想到这个？

- 当你**实现币安 WebSocket 数据源**时（第一步就是建立连接）
- 当你需要**在 SourceFunction 中初始化资源**时（连接、客户端等）
- 当你需要**处理 WebSocket 连接生命周期**时（建立、保持、关闭）
- 当你需要**理解 SourceFunction 的执行流程**时（run()方法的作用）
- 当你需要**调试连接问题**时（检查连接建立过程）


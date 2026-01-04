# WebSocket自动重连：实现重连逻辑

> ⚠️ **重要提示**：本文档中的示例代码使用 `SourceFunction`（Legacy API）实现。Flink 推荐使用新的 `Source` API。本文档主要介绍 Legacy API 的实现方式。

## 核心概念

WebSocket 连接可能因为网络问题、服务器重启等原因断开，需要实现**自动重连机制**，确保数据源能够持续运行。

### 重连策略

1. **检测连接断开**：监听 onClose、onFailure 事件
2. **延迟重连**：避免频繁重连
3. **重连次数限制**：避免无限重连

## 最小可用例子

### 使用 OkHttp

```java
public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private OkHttpClient httpClient;
    private WebSocket webSocket;
    private static final int MAX_RETRIES = 10;
    private int retryCount = 0;

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        httpClient = new OkHttpClient.Builder().build();

        // 初始连接
        connect(ctx);

        // 保持运行
        while (isRunning) {
            Thread.sleep(100);
        }
    }

    private void connect(SourceContext<Trade> ctx) {
        Request request = new Request.Builder()
            .url("wss://stream.binance.com:9443/ws/btcusdt@trade")
            .build();

        webSocket = httpClient.newWebSocket(request, new WebSocketListener() {
            @Override
            public void onMessage(WebSocket webSocket, String text) {
                try {
                    Trade trade = parseJson(text);
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(trade, trade.getTradeTime());
                    }
                    retryCount = 0;  // 重置重连计数
                } catch (Exception e) {
                    logger.error("Failed to parse message", e);
                }
            }

            @Override
            public void onFailure(WebSocket webSocket, Throwable t, Response response) {
                logger.error("WebSocket connection failed", t);
                // 触发重连
                if (isRunning && retryCount < MAX_RETRIES) {
                    scheduleReconnect(ctx);
                }
            }

            @Override
            public void onClosed(WebSocket webSocket, int code, String reason) {
                logger.info("WebSocket closed: " + reason);
                // 触发重连
                if (isRunning && retryCount < MAX_RETRIES) {
                    scheduleReconnect(ctx);
                }
            }
        });
    }

    private void scheduleReconnect(SourceContext<Trade> ctx) {
        retryCount++;
        long delay = Math.min(1000 * (1L << retryCount), 60000);  // 指数退避，最大60秒

        logger.info("Scheduling reconnect in " + delay + "ms (attempt " + retryCount + ")");

        new Thread(() -> {
            try {
                Thread.sleep(delay);
                if (isRunning) {
                    connect(ctx);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }

    @Override
    public void cancel() {
        isRunning = false;
        if (webSocket != null) {
            webSocket.close(1000, "Normal closure");
        }
    }
}
```

### 使用 Java-WebSocket

```java
public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;
    private int retryCount = 0;

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        connect(ctx);

        while (isRunning) {
            Thread.sleep(100);
        }
    }

    private void connect(SourceContext<Trade> ctx) {
        try {
            client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
                @Override
                public void onMessage(String message) {
                    try {
                        Trade trade = parseJson(message);
                        synchronized (ctx.getCheckpointLock()) {
                            ctx.collectWithTimestamp(trade, trade.getTradeTime());
                        }
                        retryCount = 0;
                    } catch (Exception e) {
                        logger.error("Failed to parse message", e);
                    }
                }

                @Override
                public void onClose(int code, String reason, boolean remote) {
                    logger.info("WebSocket closed: " + reason);
                    if (isRunning && retryCount < 10) {
                        scheduleReconnect(ctx);
                    }
                }

                @Override
                public void onError(Exception ex) {
                    logger.error("WebSocket error", ex);
                    if (isRunning && retryCount < 10) {
                        scheduleReconnect(ctx);
                    }
                }
            };

            client.connect();
        } catch (Exception e) {
            logger.error("Failed to create WebSocket client", e);
            if (isRunning && retryCount < 10) {
                scheduleReconnect(ctx);
            }
        }
    }

    private void scheduleReconnect(SourceContext<Trade> ctx) {
        retryCount++;
        long delay = Math.min(1000 * (1L << retryCount), 60000);

        logger.info("Reconnecting in " + delay + "ms (attempt " + retryCount + ")");

        new Thread(() -> {
            try {
                Thread.sleep(delay);
                if (isRunning) {
                    connect(ctx);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }
}
```

## 关键要点

1. **检测断开**：监听 onClose、onFailure 事件
2. **延迟重连**：使用指数退避算法
3. **重连限制**：避免无限重连
4. **线程安全**：使用 volatile 标志

## 什么时候你需要想到这个？

- 当你需要**处理 WebSocket 连接断开**时（网络问题、服务器重启）
- 当你需要**确保数据源持续运行**时（自动恢复连接）
- 当你需要**实现容错机制**时（重连逻辑）
- 当你需要**优化数据源稳定性**时（减少人工干预）
- 当你需要**处理生产环境的网络波动**时


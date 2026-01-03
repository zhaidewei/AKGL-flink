# 错误处理策略：确保数据不丢失

## 核心概念

实现完整的错误处理策略，确保 WebSocket 断线重连不影响数据处理，保证数据的完整性和作业的稳定性。

### 核心策略

1. **连接异常处理**：捕获WebSocket连接异常
2. **自动重连**：实现自动重连机制
3. **数据解析异常**：处理JSON解析错误
4. **状态一致性**：保证检查点一致性

## 最小可用例子

```java
public class BinanceTradeSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;
    private int retryCount = 0;
    private static final int MAX_RETRIES = 10;

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
                        Trade trade = parseTrade(message);
                        synchronized (ctx.getCheckpointLock()) {
                            ctx.collectWithTimestamp(trade, trade.getTradeTime());
                        }
                        retryCount = 0;  // 重置重连计数
                    } catch (Exception e) {
                        logger.error("Failed to parse message: " + message, e);
                        // 不抛出异常，继续处理下一条
                    }
                }

                @Override
                public void onClose(int code, String reason, boolean remote) {
                    logger.info("WebSocket closed: " + reason);
                    if (isRunning && retryCount < MAX_RETRIES) {
                        scheduleReconnect(ctx);
                    }
                }

                @Override
                public void onError(Exception ex) {
                    logger.error("WebSocket error", ex);
                    if (isRunning && retryCount < MAX_RETRIES) {
                        scheduleReconnect(ctx);
                    }
                }
            };

            client.connect();
        } catch (Exception e) {
            logger.error("Failed to create WebSocket client", e);
            if (isRunning && retryCount < MAX_RETRIES) {
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

    @Override
    public void cancel() {
        isRunning = false;
        if (client != null) {
            client.close();
        }
    }
}
```

## 错误处理层次

### 1. 连接异常

```java
try {
    client.connect();
} catch (Exception e) {
    logger.error("Connection failed", e);
    scheduleReconnect(ctx);
}
```

### 2. 消息解析异常

```java
try {
    Trade trade = parseTrade(message);
    ctx.collectWithTimestamp(trade, trade.getTradeTime());
} catch (Exception e) {
    logger.error("Parse failed", e);
    // 不抛出异常，继续处理下一条
}
```

### 3. 发送数据异常

```java
try {
    synchronized (ctx.getCheckpointLock()) {
        ctx.collectWithTimestamp(trade, trade.getTradeTime());
    }
} catch (Exception e) {
    logger.error("Send failed", e);
    // 可能需要重试或记录
}
```

## 什么时候你需要想到这个？

- 当你需要**处理WebSocket异常**时（连接、断线等）
- 当你需要**实现自动重连**时（保证数据不中断）
- 当你需要**处理数据解析错误**时（JSON解析失败）
- 当你需要**保证作业稳定性**时（异常处理）
- 当你需要**构建生产级系统**时（完整的错误处理）



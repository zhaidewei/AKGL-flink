# WebSocket连接异常处理：捕获和记录错误

> ⚠️ **重要提示**：本文档中的示例代码使用 `SourceFunction`（Legacy API）实现。Flink 推荐使用新的 `Source` API。本文档主要介绍 Legacy API 的实现方式。

## 核心概念

WebSocket 连接可能因为网络问题、服务器问题等导致异常，需要实现异常处理机制，确保数据源能够正确处理错误。

### 常见异常场景

1. **连接失败**：无法连接到币安服务器
2. **连接断开**：连接建立后意外断开
3. **消息解析失败**：JSON 格式错误
4. **网络超时**：长时间无响应

## 最小可用例子

### 使用 OkHttp

```java
public class BinanceSource implements SourceFunction<Trade> {
    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        OkHttpClient client = new OkHttpClient.Builder().build();

        Request request = new Request.Builder()
            .url("wss://stream.binance.com:9443/ws/btcusdt@trade")
            .build();

        webSocket = client.newWebSocket(request, new WebSocketListener() {
            @Override
            public void onMessage(WebSocket webSocket, String text) {
                try {
                    Trade trade = parseJson(text);
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(trade, trade.getTradeTime());
                    }
                } catch (Exception e) {
                    // 捕获解析异常
                    logger.error("Failed to parse message: " + text, e);
                    // 不抛出异常，避免整个作业失败
                }
            }

            @Override
            public void onFailure(WebSocket webSocket, Throwable t, Response response) {
                // 捕获连接失败异常
                logger.error("WebSocket connection failed", t);
                // 可以在这里触发重连逻辑
            }

            @Override
            public void onClosed(WebSocket webSocket, int code, String reason) {
                logger.info("WebSocket closed: code=" + code + ", reason=" + reason);
            }
        });

        while (isRunning) {
            Thread.sleep(100);
        }
    }
}
```

### 使用 Java-WebSocket

```java
public class BinanceSource implements SourceFunction<Trade> {
    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
            @Override
            public void onMessage(String message) {
                try {
                    Trade trade = parseJson(message);
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(trade, trade.getTradeTime());
                    }
                } catch (Exception e) {
                    logger.error("Failed to parse message", e);
                }
            }

            @Override
            public void onError(Exception ex) {
                // 捕获所有异常
                logger.error("WebSocket error", ex);
            }

            @Override
            public void onClose(int code, String reason, boolean remote) {
                logger.info("WebSocket closed: " + reason);
            }
        };

        try {
            client.connect();
        } catch (Exception e) {
            logger.error("Failed to connect to Binance", e);
            throw e;  // 连接失败应该抛出异常，让Flink知道
        }

        while (isRunning) {
            Thread.sleep(100);
        }
    }
}
```

## 异常处理策略

### 1. 连接异常

```java
try {
    client.connect();
} catch (Exception e) {
    logger.error("Failed to connect", e);
    throw e;  // 连接失败应该抛出，让Flink重试
}
```

### 2. 消息解析异常

```java
try {
    Trade trade = parseJson(message);
    ctx.collectWithTimestamp(trade, trade.getTradeTime());
} catch (Exception e) {
    logger.error("Failed to parse message", e);
    // 不抛出异常，跳过这条消息
}
```

### 3. 发送数据异常

```java
try {
    synchronized (ctx.getCheckpointLock()) {
        ctx.collectWithTimestamp(trade, trade.getTradeTime());
    }
} catch (Exception e) {
    logger.error("Failed to send data", e);
    // 可能需要重试或记录
}
```

## 日志记录

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class BinanceSource implements SourceFunction<Trade> {
    private static final Logger logger = LoggerFactory.getLogger(BinanceSource.class);

    @Override
    public void onFailure(WebSocket webSocket, Throwable t, Response response) {
        logger.error("WebSocket connection failed", t);
        if (response != null) {
            logger.error("Response code: " + response.code());
        }
    }
}
```

## 什么时候你需要想到这个？

- 当你需要**处理 WebSocket 连接异常**时（网络问题、服务器问题）
- 当你需要**处理消息解析错误**时（JSON格式错误）
- 当你需要**确保数据源的健壮性**时（异常不应该导致整个作业失败）
- 当你需要**调试数据源问题**时（通过日志定位问题）
- 当你需要**实现容错机制**时（异常处理和重连）


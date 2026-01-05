# WebSocket客户端库选择：OkHttp vs Java-WebSocket

> ✅ **重要提示**：本文档中的示例代码使用新的 `Source` API 实现。Flink 推荐使用新的 Source API，它提供了更好的性能和可扩展性。

## 核心概念

实现币安 WebSocket 数据源时，需要选择合适的 WebSocket 客户端库。常用的有 **OkHttp** 和 **Java-WebSocket**。

> **注意**：以下依赖版本仅为示例，实际使用时请检查各库的最新稳定版本。

### 两个主要选择

1. **OkHttp** - 现代化的 HTTP/WebSocket 客户端（推荐）
2. **Java-WebSocket** - 轻量级的 WebSocket 库

## OkHttp（推荐）

### 特点

- **现代化**：Google 维护，广泛使用
- **功能完整**：支持 HTTP/2、连接池、自动重连等
- **易于使用**：API 简洁
- **性能好**：经过优化

### Maven 依赖

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>  <!-- 请检查最新版本 -->
</dependency>
```

### 最小可用例子

```java
import okhttp3.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import org.apache.flink.api.connector.source.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class BinanceOkHttpReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private static final Logger logger = LoggerFactory.getLogger(BinanceOkHttpReader.class);
    private OkHttpClient client;
    private WebSocket webSocket;
    private final BlockingQueue<Trade> recordQueue = new LinkedBlockingQueue<>();
    private volatile boolean isRunning = true;

    @Override
    public void start() {
        client = new OkHttpClient.Builder()
            .pingInterval(20, TimeUnit.SECONDS)  // 保持连接
            .build();

        Request request = new Request.Builder()
            .url("wss://stream.binance.com/ws/btcusdt@trade")
            .build();

        webSocket = client.newWebSocket(request, new WebSocketListener() {
            @Override
            public void onMessage(WebSocket webSocket, String text) {
                Trade trade = parseJson(text);
                recordQueue.offer(trade);  // 非阻塞放入队列
            }

            @Override
            public void onFailure(WebSocket webSocket, Throwable t, Response response) {
                logger.error("WebSocket error", t);
            }
        });
    }

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        Trade trade = recordQueue.poll();
        if (trade != null) {
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }
        return isRunning ? InputStatus.NOTHING_AVAILABLE : InputStatus.END_OF_INPUT;
    }

    @Override
    public void close() throws Exception {
        isRunning = false;
        if (webSocket != null) {
            webSocket.close(1000, "Normal closure");
        }
    }

    // ... 其他必需的方法
}
```

## Java-WebSocket

### 特点

- **轻量级**：专门用于 WebSocket
- **简单**：API 简单直接
- **纯 Java**：不依赖其他库

### Maven 依赖

```xml
<dependency>
    <groupId>org.java-websocket</groupId>
    <artifactId>Java-WebSocket</artifactId>
    <version>1.5.4</version>  <!-- 请检查最新版本 -->
</dependency>
```

### 最小可用例子

```java
import org.java_websocket.client.WebSocketClient;
import org.java_websocket.handshake.ServerHandshake;
import java.net.URI;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import org.apache.flink.api.connector.source.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class BinanceJavaWebSocketReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private static final Logger logger = LoggerFactory.getLogger(BinanceJavaWebSocketReader.class);
    private WebSocketClient client;
    private final BlockingQueue<Trade> recordQueue = new LinkedBlockingQueue<>();
    private volatile boolean isRunning = true;

    @Override
    public void start() {
        try {
            client = new WebSocketClient(new URI("wss://stream.binance.com/ws/btcusdt@trade")) {
                @Override
                public void onMessage(String message) {
                    Trade trade = parseJson(message);
                    recordQueue.offer(trade);  // 非阻塞放入队列
                }

                @Override
                public void onOpen(ServerHandshake handshake) {
                    logger.info("WebSocket connected");
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

            client.connect();
        } catch (Exception e) {
            throw new RuntimeException("Failed to connect WebSocket", e);
        }
    }

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        Trade trade = recordQueue.poll();
        if (trade != null) {
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }
        return isRunning ? InputStatus.NOTHING_AVAILABLE : InputStatus.END_OF_INPUT;
    }

    @Override
    public void close() throws Exception {
        isRunning = false;
        if (client != null) {
            client.close();
        }
    }

    // ... 其他必需的方法
}
```

## 对比

| 特性 | OkHttp | Java-WebSocket |
|------|--------|----------------|
| 维护 | Google 维护 | 社区维护 |
| 功能 | HTTP + WebSocket | 仅 WebSocket |
| 易用性 | 较好 | 很好 |
| 性能 | 优秀 | 良好 |
| 推荐度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

## 选择建议

- **推荐 OkHttp**：如果你需要 HTTP 功能，或者想要更现代化的 API
- **选择 Java-WebSocket**：如果你只需要 WebSocket，想要更简单的 API

## 什么时候你需要想到这个？

- 当你需要**实现 WebSocket 数据源**时（币安、其他交易所）
- 当你需要**选择 WebSocket 客户端库**时
- 当你需要**处理 WebSocket 连接**时（建立、重连、错误处理）
- 当你需要**优化数据源性能**时（选择合适的库）
- 当你需要**处理币安 WebSocket API**时
- 当你使用**新 Source API** 实现 WebSocket 数据源时


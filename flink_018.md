# WebSocket消息监听：接收币安数据

> ✅ **重要提示**：本文档介绍如何在新的 `SourceReader` API 中实现 WebSocket 消息监听。Flink 推荐使用新的 Source API。

## 核心概念

实现 WebSocket 消息监听器，接收币安推送的交易数据，并发送到 Flink 流中。

### 核心流程

1. **接收消息**：WebSocket 客户端接收币安推送的 JSON 消息
2. **解析消息**：将 JSON 字符串解析为 Java 对象
3. **发送到流**：通过 SourceContext 发送到 Flink 流中

## 最小可用例子

### 使用 OkHttp

```java
import org.apache.flink.api.connector.source.*;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class BinanceSourceReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private final BlockingQueue<Trade> recordQueue = new LinkedBlockingQueue<>();

    @Override
    public void start() {
        OkHttpClient client = new OkHttpClient.Builder().build();

        Request request = new Request.Builder()
            .url("wss://stream.binance.com:9443/ws/btcusdt@trade")
            .build();

        webSocket = client.newWebSocket(request, new WebSocketListener() {
            @Override
            public void onMessage(WebSocket webSocket, String text) {
                // 1. 解析JSON消息
                Trade trade = parseJson(text);

                // 2. 放入队列（非阻塞）
                recordQueue.offer(trade);
            }

            @Override
            public void onFailure(WebSocket webSocket, Throwable t, Response response) {
                logger.error("WebSocket error", t);
            }
        });
    }

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        // 从队列中取出数据并发送
        Trade trade = recordQueue.poll();
        if (trade != null) {
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }
        return InputStatus.NOTHING_AVAILABLE;
    }

    // ... 其他必需的方法
}
```

### 使用 Java-WebSocket

```java
import org.apache.flink.api.connector.source.*;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class BinanceSourceReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private WebSocketClient client;
    private final BlockingQueue<Trade> recordQueue = new LinkedBlockingQueue<>();

    @Override
    public void start() {
        try {
            client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
                @Override
                public void onMessage(String message) {
                    // 1. 解析JSON消息
                    Trade trade = parseJson(message);

                    // 2. 放入队列（非阻塞）
                    recordQueue.offer(trade);
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
        // 从队列中取出数据并发送
        Trade trade = recordQueue.poll();
        if (trade != null) {
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }
        return InputStatus.NOTHING_AVAILABLE;
    }

    // ... 其他必需的方法
}
```

## JSON 解析示例

### 使用 Jackson

```java
import com.fasterxml.jackson.databind.ObjectMapper;

private final ObjectMapper objectMapper = new ObjectMapper();

private Trade parseJson(String json) throws Exception {
    // 解析币安Trade JSON
    JsonNode node = objectMapper.readTree(json);

    Trade trade = new Trade();
    trade.setSymbol(node.get("s").asText());              // "BTCUSDT"
    trade.setPrice(node.get("p").asDouble());            // 价格
    trade.setQuantity(node.get("q").asDouble());         // 数量
    trade.setTradeTime(node.get("T").asLong());         // 交易时间（事件时间）
    trade.setIsBuyerMaker(node.get("m").asBoolean());   // 是否为买方主动

    return trade;
}
```

### 使用 Gson

```java
import com.google.gson.Gson;
import com.google.gson.JsonObject;

private final Gson gson = new Gson();

private Trade parseJson(String json) {
    JsonObject obj = gson.fromJson(json, JsonObject.class);

    Trade trade = new Trade();
    trade.setSymbol(obj.get("s").getAsString());
    trade.setPrice(obj.get("p").getAsDouble());
    trade.setQuantity(obj.get("q").getAsDouble());
    trade.setTradeTime(obj.get("T").getAsLong());
    trade.setIsBuyerMaker(obj.get("m").getAsBoolean());

    return trade;
}
```

## 币安 Trade JSON 格式

```json
{
  "e": "trade",
  "E": 123456789,
  "s": "BTCUSDT",
  "t": 12345,
  "p": "0.001",
  "q": "100",
  "b": 88,
  "a": 50,
  "T": 123456785,
  "m": true,
  "M": true
}
```

**字段映射**：
- `s` → symbol（交易对）
- `p` → price（价格）
- `q` → quantity（数量）
- `T` → tradeTime（交易时间，事件时间）
- `m` → isBuyerMaker（是否为买方主动）

## 关键要点

1. **使用队列缓冲**：WebSocket 消息在回调中放入队列，在 `pollNext()` 中取出
2. **异常处理**：解析JSON可能失败，需要捕获异常
3. **性能考虑**：JSON解析是CPU密集型操作，考虑优化
4. **非阻塞设计**：使用 `offer()` 和 `poll()` 方法，避免阻塞

## 什么时候你需要想到这个？

- 当你需要**接收币安 WebSocket 消息**时（实现消息监听器）
- 当你需要**解析 JSON 数据**时（币安返回的是JSON格式）
- 当你需要**将外部数据发送到 Flink 流**时（通过ReaderOutput）
- 当你需要**处理实时数据流**时（币安交易数据是实时推送的）
- 当你需要**实现数据源**时（SourceReader的核心逻辑）
- 当你使用**新 Source API** 实现 WebSocket 数据源时


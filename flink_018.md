# WebSocket消息监听：接收币安数据

> ⚠️ **重要提示**：本文档介绍如何在 `SourceFunction`（Legacy API）中实现 WebSocket 消息监听。Flink 推荐使用新的 `Source` API。本文档主要介绍 Legacy API 的实现方式。

## 核心概念

实现 WebSocket 消息监听器，接收币安推送的交易数据，并发送到 Flink 流中。

### 核心流程

1. **接收消息**：WebSocket 客户端接收币安推送的 JSON 消息
2. **解析消息**：将 JSON 字符串解析为 Java 对象
3. **发送到流**：通过 SourceContext 发送到 Flink 流中

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
                // 1. 解析JSON消息
                Trade trade = parseJson(text);

                // 2. 发送到Flink流（使用检查点锁保护）
                synchronized (ctx.getCheckpointLock()) {
                    ctx.collectWithTimestamp(trade, trade.getTradeTime());
                }
            }

            @Override
            public void onFailure(WebSocket webSocket, Throwable t, Response response) {
                logger.error("WebSocket error", t);
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
                // 1. 解析JSON消息
                Trade trade = parseJson(message);

                // 2. 发送到Flink流
                synchronized (ctx.getCheckpointLock()) {
                    ctx.collectWithTimestamp(trade, trade.getTradeTime());
                }
            }

            @Override
            public void onError(Exception ex) {
                logger.error("WebSocket error", ex);
            }
        };

        client.connect();

        while (isRunning) {
            Thread.sleep(100);
        }
    }
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

1. **使用检查点锁**：在发送数据时加锁，保证容错一致性
2. **异常处理**：解析JSON可能失败，需要捕获异常
3. **性能考虑**：JSON解析是CPU密集型操作，考虑优化

## 什么时候你需要想到这个？

- 当你需要**接收币安 WebSocket 消息**时（实现消息监听器）
- 当你需要**解析 JSON 数据**时（币安返回的是JSON格式）
- 当你需要**将外部数据发送到 Flink 流**时（通过SourceContext）
- 当你需要**处理实时数据流**时（币安交易数据是实时推送的）
- 当你需要**实现数据源**时（SourceFunction的核心逻辑）


# 构建完整Source：整合WebSocket和数据模型

> ⚠️ **重要提示**：本文档介绍如何使用 `SourceFunction`（Legacy API）构建完整的数据源。Flink 推荐使用新的 `Source` API。本文档主要介绍 Legacy API 的实现方式。

## 核心概念

整合 WebSocket 连接、JSON 解析和数据模型，构建完整的币安数据源。这是实现 SourceFunction 的完整流程。

### 核心组件

1. **WebSocket连接**：连接到币安WebSocket
2. **JSON解析**：解析币安返回的JSON数据
3. **数据模型**：Trade/Ticker对象
4. **发送到流**：通过SourceContext发送到Flink流

## 最小可用例子

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.java_websocket.client.WebSocketClient;
import org.java_websocket.handshake.ServerHandshake;
import java.net.URI;

public class BinanceTradeSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        // 1. 建立WebSocket连接
        client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
            @Override
            public void onOpen(ServerHandshake handshake) {
                logger.info("WebSocket connected to Binance");
            }

            @Override
            public void onMessage(String message) {
                try {
                    // 2. 解析JSON
                    Trade trade = parseTrade(message);

                    // 3. 发送到Flink流
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

        // 4. 连接
        client.connect();

        // 5. 保持运行
        while (isRunning) {
            Thread.sleep(100);
        }
    }

    private Trade parseTrade(String json) throws Exception {
        JsonNode node = objectMapper.readTree(json);
        Trade trade = new Trade();
        trade.setSymbol(node.get("s").asText());
        trade.setPrice(node.get("p").asDouble());
        trade.setQuantity(node.get("q").asDouble());
        trade.setTradeTime(node.get("T").asLong());
        trade.setBuyerMaker(node.get("m").asBoolean());
        return trade;
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

## 使用方式

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 使用完整的SourceFunction
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Trade Source"
);

// 处理数据
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .sum("quantity")
      .print();

env.execute("Binance Trade Processing");
```

## 关键组件整合

### 1. WebSocket连接

```java
client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
    @Override
    public void onMessage(String message) {
        // 处理消息
    }
};
client.connect();
```

### 2. JSON解析

```java
private Trade parseTrade(String json) throws Exception {
    JsonNode node = objectMapper.readTree(json);
    // 解析为Trade对象
}
```

### 3. 发送到流

```java
synchronized (ctx.getCheckpointLock()) {
    ctx.collectWithTimestamp(trade, trade.getTradeTime());
}
```

## 什么时候你需要想到这个？

- 当你需要**实现完整的币安数据源**时（整合所有组件）
- 当你需要**连接WebSocket和Flink**时（完整流程）
- 当你需要**理解SourceFunction的实现**时（完整示例）
- 当你需要**调试数据源问题**时（检查各个环节）
- 当你需要**构建生产级数据源**时（完整实现）



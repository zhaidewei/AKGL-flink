# emit数据到Flink流：连接WebSocket和DataStream

## 核心概念

将解析后的币安交易数据通过 `SourceContext.collectWithTimestamp()` 发送到 Flink 流中，这是连接 WebSocket 数据源和 Flink DataStream 的关键步骤。

### 核心流程

```
币安WebSocket → JSON消息 → 解析为Trade对象 → SourceContext.collectWithTimestamp() → Flink DataStream
```

## 最小可用例子

### 完整流程

```java
public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        // 1. 建立WebSocket连接
        client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
            @Override
            public void onMessage(String message) {
                try {
                    // 2. 解析JSON为Trade对象
                    Trade trade = parseJson(message);

                    // 3. 发送到Flink流（关键步骤）
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(trade, trade.getTradeTime());
                    }
                } catch (Exception e) {
                    logger.error("Failed to process message", e);
                }
            }

            @Override
            public void onError(Exception ex) {
                logger.error("WebSocket error", ex);
            }
        };

        // 4. 连接WebSocket
        client.connect();

        // 5. 保持run()方法运行
        while (isRunning) {
            Thread.sleep(100);
        }
    }

    private Trade parseJson(String json) throws Exception {
        JsonNode node = objectMapper.readTree(json);
        Trade trade = new Trade();
        trade.setSymbol(node.get("s").asText());
        trade.setPrice(node.get("p").asDouble());
        trade.setQuantity(node.get("q").asDouble());
        trade.setTradeTime(node.get("T").asLong());
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

### 使用方式

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 使用自定义SourceFunction创建DataStream
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 对DataStream进行处理
trades.map(trade -> trade.getPrice())
      .print();

env.execute("Binance Trade Processing");
```

## 关键要点

### 1. 使用检查点锁

```java
synchronized (ctx.getCheckpointLock()) {
    ctx.collectWithTimestamp(trade, trade.getTradeTime());
}
```

**为什么需要锁？**
- 保证数据发送和状态更新是原子操作
- 确保检查点一致性
- 避免并发问题

### 2. 使用事件时间

```java
ctx.collectWithTimestamp(trade, trade.getTradeTime());
```

**为什么使用事件时间？**
- 币安交易数据包含交易发生时间
- 支持基于事件时间的窗口、Watermark等
- 可以处理乱序数据

### 3. 异常处理

```java
try {
    Trade trade = parseJson(message);
    synchronized (ctx.getCheckpointLock()) {
        ctx.collectWithTimestamp(trade, trade.getTradeTime());
    }
} catch (Exception e) {
    logger.error("Failed to process message", e);
    // 不要抛出异常，避免整个作业失败
}
```

## 数据流转

```
币安服务器
    ↓ (WebSocket推送)
JSON消息: {"s":"BTCUSDT","p":"50000","q":"0.1","T":1234567890}
    ↓ (parseJson)
Trade对象: Trade{symbol="BTCUSDT", price=50000.0, quantity=0.1, tradeTime=1234567890}
    ↓ (collectWithTimestamp)
Flink DataStream<Trade>
    ↓ (后续处理)
map, filter, keyBy, window等操作
```

## 什么时候你需要想到这个？

- 当你需要**将 WebSocket 数据发送到 Flink 流**时（连接外部数据源和Flink）
- 当你需要**实现完整的数据源**时（从连接到发送的完整流程）
- 当你需要**理解 SourceFunction 的数据流转**时（数据如何进入Flink）
- 当你需要**调试数据源问题**时（检查数据是否正确发送）
- 当你需要**优化数据源性能**时（减少发送延迟）


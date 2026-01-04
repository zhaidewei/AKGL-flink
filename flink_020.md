# emit数据到Flink流：连接WebSocket和DataStream

> ✅ **重要提示**：本文档介绍如何使用 `ReaderOutput.collect()`（新 Source API）将数据发送到 Flink 流。Flink 推荐使用新的 Source API。

## 核心概念

将解析后的币安交易数据通过 `ReaderOutput.collect()` 发送到 Flink 流中，这是连接 WebSocket 数据源和 Flink DataStream 的关键步骤。

### 核心流程

```
币安WebSocket → JSON消息 → 解析为Trade对象 → 放入队列 → pollNext() → ReaderOutput.collect() → Flink DataStream
```

## 最小可用例子

### 完整流程

```java
import org.apache.flink.api.connector.source.*;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class BinanceSourceReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private WebSocketClient client;
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final BlockingQueue<Trade> recordQueue = new LinkedBlockingQueue<>();
    private volatile boolean isRunning = true;

    @Override
    public void start() {
        // 1. 建立WebSocket连接
        try {
            client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
                @Override
                public void onMessage(String message) {
                    try {
                        // 2. 解析JSON为Trade对象
                        Trade trade = parseJson(message);

                        // 3. 放入队列（非阻塞）
                        recordQueue.offer(trade);
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
        } catch (Exception e) {
            throw new RuntimeException("Failed to connect WebSocket", e);
        }
    }

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        // 5. 从队列中取出数据
        Trade trade = recordQueue.poll();

        if (trade != null) {
            // 6. 发送到Flink流（关键步骤）
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }

        return isRunning ? InputStatus.NOTHING_AVAILABLE : InputStatus.END_OF_INPUT;
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
    public void close() throws Exception {
        isRunning = false;
        if (client != null) {
            client.close();
        }
    }

    // ... 其他必需的方法
}
```

### 使用方式

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 1. 创建 Source
BinanceWebSocketSource source = new BinanceWebSocketSource("btcusdt");

// 2. 创建 WatermarkStrategy（用于事件时间处理）
WatermarkStrategy<Trade> watermarkStrategy = WatermarkStrategy
    .<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime());

// 3. 使用新 Source API 创建 DataStream
DataStream<Trade> trades = env.fromSource(
    source,
    watermarkStrategy,
    "Binance Trade Source"
);

// 4. 对DataStream进行处理
trades.map(trade -> trade.getPrice())
      .print();

env.execute("Binance Trade Processing");
```

## 关键要点

### 1. 使用队列缓冲数据

```java
// WebSocket 消息回调中放入队列
client.onMessage(message -> {
    Trade trade = parseJson(message);
    recordQueue.offer(trade);  // 非阻塞放入队列
});

// pollNext() 中从队列取出并发送
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
    Trade trade = recordQueue.poll();  // 非阻塞取出
    if (trade != null) {
        output.collect(trade);
        return InputStatus.MORE_AVAILABLE;
    }
    return InputStatus.NOTHING_AVAILABLE;
}
```

### 2. 使用事件时间（通过 WatermarkStrategy）

```java
// 在创建 DataStream 时指定 WatermarkStrategy
WatermarkStrategy<Trade> watermarkStrategy = WatermarkStrategy
    .<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime());

DataStream<Trade> trades = env.fromSource(source, watermarkStrategy, "Source");
```

### 3. 异常处理

```java
try {
    Trade trade = parseJson(message);
    recordQueue.offer(trade);
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
    ↓ (offer to queue)
BlockingQueue<Trade>
    ↓ (poll from queue)
pollNext(ReaderOutput)
    ↓ (collect)
Flink DataStream<Trade>
    ↓ (后续处理)
map, filter, keyBy, window等操作
```

## 与 Legacy API 的对比

| 特性 | Legacy SourceContext | 新 ReaderOutput |
|------|---------------------|-----------------|
| 调用位置 | run() 方法中 | pollNext() 方法中 |
| 是否需要锁 | 需要（getCheckpointLock()） | 不需要 |
| 阻塞性 | 可能阻塞 | 非阻塞 |
| 时间戳处理 | collectWithTimestamp() | WatermarkStrategy |
| 推荐度 | ⚠️ 不推荐 | ✅ 推荐 |

## 什么时候你需要想到这个？

- 当你需要**将 WebSocket 数据发送到 Flink 流**时（连接外部数据源和Flink）
- 当你需要**实现完整的数据源**时（从连接到发送的完整流程）
- 当你需要**理解 SourceReader 的数据流转**时（数据如何进入Flink）
- 当你需要**调试数据源问题**时（检查数据是否正确发送）
- 当你需要**优化数据源性能**时（减少发送延迟）
- 当你使用**新 Source API** 实现 WebSocket 数据源时

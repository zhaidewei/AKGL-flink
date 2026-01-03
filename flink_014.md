# SourceContext.collectWithTimestamp()：带时间戳的数据

## 核心概念

**`collectWithTimestamp(T element, long timestamp)`** 用于发送**带时间戳**的数据到 Flink 流中。这是事件时间（Event Time）处理的关键方法。

### 类比理解

这就像：
- **带时间戳的日志**：不仅记录内容，还记录时间
- **时间序列数据**：每个数据点都有对应的时间
- **事件溯源**：记录事件发生的时间

### 核心特点

1. **指定事件时间**：数据本身携带的时间戳
2. **用于事件时间处理**：支持基于事件时间的窗口、Watermark 等
3. **处理乱序数据**：Flink 可以根据时间戳处理乱序到达的数据

## 源码位置

collectWithTimestamp() 方法定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java)

### 方法签名

```java
public interface SourceContext<T> {
    /**
     * 发送一个元素到流中，并指定时间戳
     * @param element 要发送的元素
     * @param timestamp 时间戳（毫秒，从1970-01-01 00:00:00 UTC开始）
     */
    void collectWithTimestamp(T element, long timestamp);
}
```

## 最小可用例子

### 币安WebSocket示例（事件时间）

```java
public class BinanceSource implements SourceFunction<Trade> {
    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        WebSocketClient client = new WebSocketClient("wss://stream.binance.com/ws/btcusdt@trade");

        client.onMessage(message -> {
            Trade trade = parseJson(message);

            // 获取币安交易数据中的事件时间（交易发生时间）
            long eventTime = trade.getTradeTime();  // 币安返回的时间戳（毫秒）

            // 使用检查点锁保护
            synchronized (ctx.getCheckpointLock()) {
                // 发送带时间戳的数据
                ctx.collectWithTimestamp(trade, eventTime);
            }
        });

        while (isRunning) {
            Thread.sleep(100);
        }
    }
}
```

### Trade 数据类示例

```java
public class Trade implements Serializable {
    private String symbol;        // 交易对，如 "BTCUSDT"
    private double price;         // 价格
    private double quantity;     // 数量
    private long tradeTime;       // 交易时间（事件时间）

    // getter/setter...
    public long getTradeTime() {
        return tradeTime;
    }
}
```

## 时间戳格式

时间戳必须是**毫秒数**，从 1970-01-01 00:00:00 UTC 开始：

```java
// 方式1：使用 System.currentTimeMillis()
long timestamp = System.currentTimeMillis();

// 方式2：从数据中提取（币安返回的时间戳）
long timestamp = trade.getTradeTime();

// 方式3：解析时间字符串
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
long timestamp = sdf.parse("2024-01-01 12:00:00").getTime();
```

## 使用场景

### 场景1：事件时间处理（币安交易数据）

```java
// 币安交易数据包含交易时间，使用事件时间处理
ctx.collectWithTimestamp(trade, trade.getTradeTime());
```

### 场景2：处理乱序数据

```java
// 数据可能乱序到达，但包含事件时间，Flink可以正确处理
ctx.collectWithTimestamp(data, data.getEventTime());
```

## 与 collect() 的区别

| 方法 | 时间戳 | 适用场景 |
|------|--------|---------|
| `collect()` | 无（处理时间） | 不需要事件时间 |
| `collectWithTimestamp()` | 有（事件时间） | 需要事件时间处理 |

### 示例对比

```java
// 使用 collect() - 处理时间模式
ctx.collect(trade);  // Flink使用当前系统时间

// 使用 collectWithTimestamp() - 事件时间模式
ctx.collectWithTimestamp(trade, trade.getTradeTime());  // 使用交易实际发生时间
```

## 注意事项

1. **时间戳单位**：必须是毫秒（不是秒）
2. **时间戳来源**：应该来自数据本身（事件时间），不是当前系统时间
3. **建议使用检查点锁**：保证容错一致性

## 什么时候你需要想到这个？

- 当你需要**事件时间处理**时（窗口、Watermark等）
- 当你处理**币安交易数据**时（数据包含交易时间）
- 当你需要处理**乱序数据**时（基于事件时间排序）
- 当你需要**时间相关的计算**时（如计算1分钟内的交易量）
- 当你实现**实时数据源**时（数据本身包含时间戳）


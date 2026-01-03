# SourceContext：数据发送的上下文对象

## 核心概念

**SourceContext** 是 SourceFunction 中用于**发送数据到 Flink 流**的上下文对象。它提供了发送数据、设置时间戳、发送 Watermark 等方法。

### 类比理解

这就像：
- **输出流（OutputStream）**：提供 `write()` 方法写入数据
- **Kafka Producer**：提供 `send()` 方法发送消息
- **数据库连接**：提供 `execute()` 方法执行SQL

### 核心方法

SourceContext 提供了以下关键方法：
1. **`collect(T element)`** - 发送数据（不带时间戳）
2. **`collectWithTimestamp(T element, long timestamp)`** - 发送数据（带时间戳）
3. **`emitWatermark(Watermark mark)`** - 发送 Watermark
4. **`getCheckpointLock()`** - 获取检查点锁（用于容错）

## 源码位置

SourceContext 接口定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java)

### 接口定义（伪代码）

```java
public interface SourceFunction<T> {
    interface SourceContext<T> {
        // 发送数据（不带时间戳）
        void collect(T element);

        // 发送数据（带时间戳，用于事件时间处理）
        void collectWithTimestamp(T element, long timestamp);

        // 发送Watermark
        void emitWatermark(Watermark mark);

        // 获取检查点锁（用于容错一致性）
        Object getCheckpointLock();

        // 标记源为暂时空闲
        void markAsTemporarilyIdle();

        // 关闭上下文
        void close();
    }
}
```

## 最小可用例子

```java
public class MySource implements SourceFunction<String> {
    @Override
    public void run(SourceContext<String> ctx) throws Exception {
        // 方式1：发送不带时间戳的数据
        ctx.collect("hello");

        // 方式2：发送带时间戳的数据（用于事件时间处理）
        long timestamp = System.currentTimeMillis();
        ctx.collectWithTimestamp("world", timestamp);

        // 方式3：发送Watermark
        ctx.emitWatermark(new Watermark(timestamp));

        // 方式4：在检查点锁保护下发送数据（推荐，保证容错一致性）
        synchronized (ctx.getCheckpointLock()) {
            ctx.collect("flink");
        }
    }
}
```

## 关键方法详解

### 1. collect() - 发送数据

```java
// 最简单的方式，发送数据到流中
ctx.collect(trade);
```

**使用场景**：处理时间（Processing Time）模式，不需要事件时间。

### 2. collectWithTimestamp() - 发送带时间戳的数据

```java
// 发送数据并指定时间戳（事件时间）
long eventTime = trade.getTimestamp();
ctx.collectWithTimestamp(trade, eventTime);
```

**使用场景**：事件时间（Event Time）模式，需要基于数据本身的时间戳处理。

### 3. getCheckpointLock() - 检查点锁

```java
// 在锁保护下发送数据，保证检查点一致性
synchronized (ctx.getCheckpointLock()) {
    ctx.collect(trade);
    // 更新内部状态（如果有）
    updateInternalState();
}
```

**为什么需要锁？**：保证数据发送和状态更新是原子操作，避免检查点时数据不一致。

## 币安WebSocket完整示例

```java
public class BinanceSource implements SourceFunction<Trade> {
    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        WebSocketClient client = new WebSocketClient("wss://stream.binance.com/ws/btcusdt@trade");

        client.onMessage(message -> {
            try {
                Trade trade = parseJson(message);

                // 使用检查点锁保护（推荐）
                synchronized (ctx.getCheckpointLock()) {
                    // 发送带时间戳的数据（币安交易数据包含时间戳）
                    ctx.collectWithTimestamp(trade, trade.getEventTime());
                }
            } catch (Exception e) {
                logger.error("Failed to process message", e);
            }
        });

        while (isRunning) {
            Thread.sleep(100);
        }
    }
}
```

## 什么时候你需要想到这个？

- 当你**在 SourceFunction 中发送数据**时（必须使用 SourceContext）
- 当你需要**发送带时间戳的数据**时（事件时间处理）
- 当你需要**保证容错一致性**时（使用 getCheckpointLock()）
- 当你实现**自定义数据源**时（WebSocket、数据库等）
- 当你需要理解 Flink 的**数据发送机制**时


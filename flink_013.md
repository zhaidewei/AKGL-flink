# SourceContext.collect()：发送数据到流中

## 核心概念

**`collect(T element)`** 是 SourceContext 中最基本的方法，用于将数据元素发送到 Flink 流中。这是最常用的数据发送方式。

### 类比理解

这就像：
- **打印语句**：`System.out.println(data)` - 输出数据
- **队列的 offer()**：将数据放入队列
- **流的写入**：将数据写入输出流

### 核心特点

1. **最简单的方式**：不需要指定时间戳
2. **适用于处理时间模式**：Flink 会自动分配处理时间
3. **线程安全**：可以在多线程环境中调用（但建议加锁）

## 源码位置

collect() 方法定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java)

### 方法签名

```java
public interface SourceContext<T> {
    /**
     * 发送一个元素到流中（不带时间戳）
     * @param element 要发送的元素
     */
    void collect(T element);
}
```

## 最小可用例子

### 简单示例

```java
public class SimpleSource implements SourceFunction<String> {
    @Override
    public void run(SourceContext<String> ctx) throws Exception {
        // 发送字符串数据
        ctx.collect("hello");
        ctx.collect("world");
        ctx.collect("flink");
    }
}
```

### 币安WebSocket示例

```java
public class BinanceSource implements SourceFunction<Trade> {
    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        WebSocketClient client = new WebSocketClient("wss://stream.binance.com/ws/btcusdt@trade");

        client.onMessage(message -> {
            Trade trade = parseJson(message);

            // 使用检查点锁保护（推荐做法）
            synchronized (ctx.getCheckpointLock()) {
                ctx.collect(trade);  // 发送交易数据
            }
        });

        while (isRunning) {
            Thread.sleep(100);
        }
    }
}
```

## 使用场景

### 场景1：处理时间模式（不需要事件时间）

```java
// 如果不需要基于事件时间处理，使用collect()即可
ctx.collect(data);
```

### 场景2：数据源不提供时间戳

```java
// 如果数据源（如某些API）不提供时间戳，使用collect()
ctx.collect(apiResponse);
```

## 注意事项

1. **建议使用检查点锁**：在多线程或需要容错的场景下
2. **不要存储 SourceContext**：只在 run() 方法中使用
3. **异常处理**：collect() 可能抛出异常，需要处理

### 推荐模式

```java
synchronized (ctx.getCheckpointLock()) {
    ctx.collect(data);
}
```

## 与 collectWithTimestamp() 的区别

| 方法 | 时间戳 | 适用场景 |
|------|--------|---------|
| `collect()` | 无（使用处理时间） | 处理时间模式 |
| `collectWithTimestamp()` | 有（事件时间） | 事件时间模式 |

## 什么时候你需要想到这个？

- 当你**在 SourceFunction 中发送数据**时（最基本的方法）
- 当你使用**处理时间模式**时（不需要事件时间）
- 当你需要**简单快速地发送数据**时（不需要时间戳）
- 当你实现**简单的数据源**时（如生成测试数据）
- 当你学习 Flink 的**数据发送机制**时（从最简单的方法开始）


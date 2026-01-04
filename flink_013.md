# SourceReader.emitRecord()：发送数据到流中

> ✅ **重要提示**：在新 Source API 中，使用 `ReaderOutput.collect()` 发送数据到 Flink 流中。它替代了 Legacy API 中的 `SourceContext.collect()` 方法。

## 核心概念

**`ReaderOutput.collect(T element)`** 是新 Source API 中用于将数据元素发送到 Flink 流中的方法。这是最常用的数据发送方式。

### 类比理解

这就像：
- **打印语句**：`System.out.println(data)` - 输出数据
- **队列的 offer()**：将数据放入队列
- **流的写入**：将数据写入输出流

### 核心特点

1. **最简单的方式**：不需要指定时间戳
2. **适用于处理时间模式**：Flink 会自动分配处理时间
3. **非阻塞**：可以在 `pollNext()` 中调用

## 源码位置

ReaderOutput.collect() 方法定义在：
[flink-core/src/main/java/org/apache/flink/api/connector/source/ReaderOutput.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/connector/source/ReaderOutput.java)

### 方法签名

```java
public interface ReaderOutput<T> {
    /**
     * 发送一个元素到流中（不带时间戳）
     * @param record 要发送的元素
     */
    void collect(T record);
}
```

## 最小可用例子

### 简单示例

```java
public class SimpleSourceReader implements SourceReader<String, MySplit> {
    @Override
    public InputStatus pollNext(ReaderOutput<String> output) throws Exception {
        // 发送字符串数据
        output.collect("hello");
        output.collect("world");
        output.collect("flink");
        return InputStatus.MORE_AVAILABLE;
    }
}
```

### 币安 WebSocket 示例

```java
public class BinanceWebSocketReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private final BlockingQueue<Trade> recordQueue = new LinkedBlockingQueue<>();

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        // 1. 从队列中获取数据（非阻塞）
        Trade trade = recordQueue.poll();

        if (trade != null) {
            // 2. 发送数据到 Flink 流
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }

        return InputStatus.NOTHING_AVAILABLE;
    }

    // WebSocket 消息回调
    private void onWebSocketMessage(String message) {
        Trade trade = parseJson(message);
        recordQueue.offer(trade);  // 放入队列
    }
}
```

## 使用场景

### 场景1：处理时间模式（不需要事件时间）

```java
// 如果不需要基于事件时间处理，使用 collect() 即可
output.collect(data);
```

### 场景2：数据源不提供时间戳

```java
// 如果数据源（如某些API）不提供时间戳，使用 collect()
output.collect(apiResponse);
```

## 注意事项

1. **在 pollNext() 中调用**：只能在 `pollNext()` 方法中使用
2. **非阻塞**：方法本身是非阻塞的
3. **不需要锁**：不需要像 Legacy API 那样使用检查点锁

## 与 Legacy SourceContext.collect() 的区别

| 特性 | Legacy SourceContext.collect() | 新 ReaderOutput.collect() |
|------|-------------------------------|---------------------------|
| 调用位置 | run() 方法中 | pollNext() 方法中 |
| 是否需要锁 | 需要（getCheckpointLock()） | 不需要 |
| 阻塞性 | 可能阻塞 | 非阻塞 |
| 推荐度 | ⚠️ 不推荐 | ✅ 推荐 |

### 示例对比

```java
// Legacy API（不推荐）
@Override
public void run(SourceContext<Trade> ctx) throws Exception {
    synchronized (ctx.getCheckpointLock()) {
        ctx.collect(trade);  // 需要加锁
    }
}

// 新 API（推荐）
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
    output.collect(trade);  // 不需要加锁
    return InputStatus.MORE_AVAILABLE;
}
```

## 什么时候你需要想到这个？

- 当你**在 SourceReader 中发送数据**时（最基本的方法）
- 当你使用**处理时间模式**时（不需要事件时间）
- 当你需要**简单快速地发送数据**时（不需要时间戳）
- 当你实现**简单的数据源**时（如生成测试数据）
- 当你学习 Flink 的**新数据发送机制**时（从最简单的方法开始）

# SourceReader：数据读取的核心组件

> ✅ **重要提示**：`SourceReader` 是 Flink 新 Source API 的核心组件，用于实际读取数据。它替代了 Legacy API 中的 `SourceFunction.run()` 方法。

## 核心概念

**SourceReader** 是新 Source API 中**实际读取数据**的组件。它负责从数据源（如币安 WebSocket）读取数据，并通过 `ReaderOutput` 发送到 Flink 流中。

### 类比理解

这就像：
- **线程的 run() 方法**：定义了要执行的任务
- **Kafka Consumer 的 poll() 循环**：持续从 Kafka 拉取消息
- **事件循环**：持续监听事件并处理

### 核心特点

1. **非阻塞设计**：`pollNext()` 方法必须是非阻塞的
2. **异步支持**：通过 `isAvailable()` 返回 Future 来支持异步操作
3. **分片管理**：可以处理多个数据分片
4. **检查点支持**：通过 `snapshotState()` 支持检查点

## 源码位置

SourceReader 接口定义在：
[flink-core/src/main/java/org/apache/flink/api/connector/source/SourceReader.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/connector/source/SourceReader.java)

### 接口定义（关键方法）

```java
public interface SourceReader<T, SplitT extends SourceSplit>
        extends AutoCloseable, CheckpointListener {

    /**
     * 启动读取器
     */
    void start();

    /**
     * 轮询下一个可用记录（非阻塞）
     * @param output 用于发送数据的输出对象
     * @return 输入状态（是否有更多数据）
     */
    InputStatus pollNext(ReaderOutput<T> output) throws Exception;

    /**
     * 检查点状态快照
     * @return 当前分片状态列表
     */
    List<SplitT> snapshotState(long checkpointId);

    /**
     * 返回一个 Future，表示数据是否可用
     */
    CompletableFuture<Void> isAvailable();

    /**
     * 添加分片（由 SplitEnumerator 分配）
     */
    void addSplits(List<SplitT> splits);

    /**
     * 通知不再有更多分片
     */
    void notifyNoMoreSplits();

    /**
     * 关闭读取器，释放资源
     */
    void close() throws Exception;
}
```

## 最小可用例子

### 币安 WebSocket SourceReader（简化版）

```java
import org.apache.flink.api.connector.source.*;
import org.apache.flink.core.io.InputStatus;
import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class BinanceWebSocketReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private final SourceReaderContext context;
    private final String symbol;
    private WebSocketClient client;
    private final BlockingQueue<Trade> recordQueue = new LinkedBlockingQueue<>();
    private volatile boolean isRunning = true;

    public BinanceWebSocketReader(SourceReaderContext context, String symbol) {
        this.context = context;
        this.symbol = symbol;
    }

    @Override
    public void start() {
        // 1. 建立 WebSocket 连接
        try {
            client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/" + symbol + "@trade")) {
                @Override
                public void onMessage(String message) {
                    try {
                        // 2. 解析 JSON 消息
                        Trade trade = parseJson(message);
                        // 3. 放入队列（非阻塞）
                        recordQueue.offer(trade);
                    } catch (Exception e) {
                        context.sendSourceEventToCoordinator(
                            new SourceEvent() {
                                // 发送错误事件
                            }
                        );
                    }
                }

                @Override
                public void onError(Exception ex) {
                    // 处理错误
                }
            };
            client.connect();
        } catch (Exception e) {
            throw new RuntimeException("Failed to connect WebSocket", e);
        }
    }

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        // 1. 从队列中获取数据（非阻塞）
        Trade trade = recordQueue.poll();

        if (trade != null) {
            // 2. 发送数据到 Flink 流
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;  // 还有更多数据
        }

        // 3. 如果没有数据，返回状态
        if (isRunning) {
            return InputStatus.NOTHING_AVAILABLE;  // 暂时没有数据，但还在运行
        } else {
            return InputStatus.END_OF_INPUT;  // 输入结束
        }
    }

    @Override
    public List<BinanceWebSocketSplit> snapshotState(long checkpointId) {
        // 返回当前分片状态（对于 WebSocket，可能不需要状态）
        return Collections.emptyList();
    }

    @Override
    public CompletableFuture<Void> isAvailable() {
        // 如果队列中有数据，立即返回完成的 Future
        if (!recordQueue.isEmpty()) {
            return CompletableFuture.completedFuture(null);
        }
        // 否则返回一个等待的 Future（当有新数据时完成）
        CompletableFuture<Void> future = new CompletableFuture<>();
        // 这里可以设置一个监听器，当有新数据时完成 Future
        // 简化示例：返回一个立即完成的 Future（实际应该等待数据）
        return CompletableFuture.completedFuture(null);
    }

    @Override
    public void addSplits(List<BinanceWebSocketSplit> splits) {
        // 对于 WebSocket，通常只有一个分片
        // 可以在这里处理分片分配
    }

    @Override
    public void notifyNoMoreSplits() {
        // 通知不再有更多分片
    }

    @Override
    public void close() throws Exception {
        isRunning = false;
        if (client != null) {
            client.close();
        }
    }

    private Trade parseJson(String json) throws Exception {
        // JSON 解析逻辑
        // ...
        return trade;
    }
}
```

## 关键要点

### 1. pollNext() 必须是非阻塞的

```java
// ✅ 正确：使用队列，非阻塞
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
    Trade trade = recordQueue.poll();  // 非阻塞
    if (trade != null) {
        output.collect(trade);
        return InputStatus.MORE_AVAILABLE;
    }
    return InputStatus.NOTHING_AVAILABLE;
}

// ❌ 错误：阻塞操作
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
    Trade trade = recordQueue.take();  // 阻塞！会导致问题
    output.collect(trade);
    return InputStatus.MORE_AVAILABLE;
}
```

### 2. 使用队列缓冲数据

```java
// WebSocket 消息在回调中放入队列
client.onMessage(message -> {
    Trade trade = parseJson(message);
    recordQueue.offer(trade);  // 非阻塞放入队列
});

// pollNext() 从队列中取出数据
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) {
    Trade trade = recordQueue.poll();  // 非阻塞取出
    if (trade != null) {
        output.collect(trade);
    }
    // ...
}
```

### 3. 使用 isAvailable() 支持异步

```java
@Override
public CompletableFuture<Void> isAvailable() {
    if (!recordQueue.isEmpty()) {
        return CompletableFuture.completedFuture(null);
    }
    // 返回一个 Future，当有新数据时完成
    CompletableFuture<Void> future = new CompletableFuture<>();
    // 设置监听器，当队列有新数据时完成 Future
    return future;
}
```

## 与 Legacy SourceFunction.run() 的对比

| 特性 | Legacy SourceFunction.run() | 新 SourceReader |
|------|----------------------------|-----------------|
| 执行模式 | 阻塞循环 | 非阻塞轮询 |
| 数据发送 | SourceContext.collect() | ReaderOutput.collect() |
| 线程模型 | 单线程阻塞 | 支持异步 |
| 分片支持 | 不支持 | 支持 |
| 推荐度 | ⚠️ 不推荐 | ✅ 推荐 |

## 常见错误

### 错误1：pollNext() 中执行阻塞操作

```java
// ❌ 错误：在 pollNext() 中阻塞
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
    Trade trade = client.receive();  // 阻塞操作！
    output.collect(trade);
    return InputStatus.MORE_AVAILABLE;
}

// ✅ 正确：使用队列，非阻塞
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
    Trade trade = recordQueue.poll();  // 非阻塞
    if (trade != null) {
        output.collect(trade);
        return InputStatus.MORE_AVAILABLE;
    }
    return InputStatus.NOTHING_AVAILABLE;
}
```

### 错误2：忘记实现 isAvailable()

```java
// ❌ 错误：返回 null
@Override
public CompletableFuture<Void> isAvailable() {
    return null;  // 会导致 NullPointerException
}

// ✅ 正确：返回有效的 Future
@Override
public CompletableFuture<Void> isAvailable() {
    if (!recordQueue.isEmpty()) {
        return CompletableFuture.completedFuture(null);
    }
    return new CompletableFuture<>();  // 等待数据
}
```

### 错误3：在 start() 中执行长时间操作

```java
// ❌ 错误：在 start() 中阻塞
@Override
public void start() {
    while (true) {  // 阻塞，导致 start() 无法返回
        // ...
    }
}

// ✅ 正确：在 start() 中初始化，实际读取在 pollNext() 中
@Override
public void start() {
    client = new WebSocketClient(...);
    client.connect();  // 快速初始化
    // 实际读取在 pollNext() 中通过队列进行
}
```

## 什么时候你需要想到这个？

- 当你需要**实现新 Source API 的数据读取**时（核心组件）
- 当你需要**从外部系统读取数据**时（WebSocket、数据库、文件等）
- 当你需要理解 Flink 的**非阻塞数据读取机制**时
- 当你需要**优化数据源性能**时（异步、非阻塞）
- 当你需要**支持数据分片**时（新 API 支持分片）

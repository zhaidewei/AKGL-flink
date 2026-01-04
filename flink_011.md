# SourceReader的启动和关闭：生命周期管理

> ✅ **重要提示**：`SourceReader` 的 `start()` 和 `close()` 方法用于管理数据源的生命周期。它们替代了 Legacy API 中的 `SourceFunction.run()` 和 `cancel()` 方法。

## 核心概念

**`start()`** 和 **`close()``** 方法用于管理 SourceReader 的**生命周期**。`start()` 在读取器启动时调用，用于初始化资源；`close()` 在读取器关闭时调用，用于清理资源。

### 类比理解

这就像：
- **线程的 start() 和 close()**：启动和关闭线程
- **资源的 open() 和 close()**：打开和关闭资源
- **服务的启动和停止**：管理服务的生命周期

### 核心职责

1. **`start()`**：初始化资源（建立连接、启动线程等）
2. **`close()`**：清理资源（关闭连接、释放资源等）

## 源码位置

SourceReader 的 start() 和 close() 方法定义在：
[flink-core/src/main/java/org/apache/flink/api/connector/source/SourceReader.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/connector/source/SourceReader.java)

### 方法签名

```java
public interface SourceReader<T, SplitT extends SourceSplit>
        extends AutoCloseable, CheckpointListener {

    /**
     * 启动读取器
     * 在读取器开始工作前调用，用于初始化资源
     */
    void start();

    /**
     * 关闭读取器，释放资源
     * 实现 AutoCloseable 接口
     */
    void close() throws Exception;
}
```

## 最小可用例子

### 币安 WebSocket SourceReader 示例

```java
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
        // 1. 创建 WebSocket 客户端
        try {
            client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/" + symbol + "@trade")) {
                @Override
                public void onMessage(String message) {
                    try {
                        Trade trade = parseJson(message);
                        recordQueue.offer(trade);  // 非阻塞放入队列
                    } catch (Exception e) {
                        logger.error("Failed to parse message", e);
                    }
                }

                @Override
                public void onOpen(ServerHandshake handshake) {
                    logger.info("WebSocket connected");
                }

                @Override
                public void onClose(int code, String reason, boolean remote) {
                    logger.info("WebSocket closed: " + reason);
                    isRunning = false;
                }

                @Override
                public void onError(Exception ex) {
                    logger.error("WebSocket error", ex);
                }
            };

            // 2. 建立连接
            client.connect();

            // 3. 等待连接建立（可选）
            while (!client.isOpen() && isRunning) {
                Thread.sleep(100);
            }

            logger.info("Binance WebSocket reader started");
        } catch (Exception e) {
            throw new RuntimeException("Failed to start WebSocket reader", e);
        }
    }

    @Override
    public void close() throws Exception {
        // 1. 设置停止标志
        isRunning = false;

        // 2. 关闭 WebSocket 连接
        if (client != null) {
            try {
                client.close();
                logger.info("WebSocket connection closed");
            } catch (Exception e) {
                logger.error("Error closing WebSocket", e);
            }
        }

        // 3. 清理队列（可选）
        recordQueue.clear();

        logger.info("Binance WebSocket reader closed");
    }

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        Trade trade = recordQueue.poll();
        if (trade != null) {
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }
        return isRunning ? InputStatus.NOTHING_AVAILABLE : InputStatus.END_OF_INPUT;
    }

    // ... 其他方法
}
```

## 关键要点

### 1. start() 中初始化资源

```java
@Override
public void start() {
    // ✅ 正确：在 start() 中初始化连接
    client = new WebSocketClient(...);
    client.connect();

    // ❌ 错误：不要在 start() 中执行长时间阻塞操作
    // while (true) { ... }  // 会导致 start() 无法返回
}
```

### 2. close() 中清理资源

```java
@Override
public void close() throws Exception {
    // ✅ 正确：清理所有资源
    isRunning = false;
    if (client != null) {
        client.close();
    }
    recordQueue.clear();
}
```

### 3. 使用 volatile 标志

```java
// ✅ 正确：使用 volatile 保证可见性
private volatile boolean isRunning = true;

@Override
public void close() throws Exception {
    isRunning = false;  // 其他线程可以看到这个变化
}
```

## 与 Legacy SourceFunction 的对比

| 特性 | Legacy SourceFunction | 新 SourceReader |
|------|----------------------|-----------------|
| 启动方法 | run() | start() |
| 停止方法 | cancel() | close() |
| 执行模式 | 阻塞循环 | 非阻塞轮询 |
| 资源管理 | 手动管理 | 自动管理（AutoCloseable） |
| 推荐度 | ⚠️ 不推荐 | ✅ 推荐 |

## 常见错误

### 错误1：在 start() 中执行阻塞操作

```java
// ❌ 错误：在 start() 中阻塞
@Override
public void start() {
    while (true) {  // 阻塞，导致 start() 无法返回
        // ...
    }
}

// ✅ 正确：在 start() 中快速初始化
@Override
public void start() {
    client = new WebSocketClient(...);
    client.connect();  // 快速初始化，实际读取在 pollNext() 中
}
```

### 错误2：忘记关闭资源

```java
// ❌ 错误：没有关闭连接
@Override
public void close() throws Exception {
    isRunning = false;
    // 忘记关闭 client，资源泄漏
}

// ✅ 正确：关闭所有资源
@Override
public void close() throws Exception {
    isRunning = false;
    if (client != null) {
        client.close();
    }
}
```

### 错误3：不使用 volatile

```java
// ❌ 错误：isRunning 不是 volatile
private boolean isRunning = true;  // 可能导致可见性问题

// ✅ 正确：使用 volatile
private volatile boolean isRunning = true;
```

### 错误4：在 close() 中抛出未捕获的异常

```java
// ❌ 错误：异常导致资源无法清理
@Override
public void close() throws Exception {
    client.close();  // 可能抛出异常，导致后续清理无法执行
    recordQueue.clear();
}

// ✅ 正确：捕获异常，确保所有资源都被清理
@Override
public void close() throws Exception {
    isRunning = false;
    if (client != null) {
        try {
            client.close();
        } catch (Exception e) {
            logger.error("Error closing WebSocket", e);
        }
    }
    recordQueue.clear();
}
```

## 什么时候你需要想到这个？

- 当你**实现 SourceReader** 时（必须实现这两个方法）
- 当你需要**初始化数据源资源**时（连接、客户端等）
- 当你需要**优雅关闭数据源**时（避免资源泄漏）
- 当你需要理解 Flink 的**数据源生命周期**时
- 当你需要**处理资源管理**时（连接、线程等）

# SourceFunction接口：自定义数据源的基础

> ⚠️ **重要提示**：`SourceFunction` 是 Flink 的 **Legacy API**（遗留 API），位于 `legacy` 包下，并被标记为 `@Internal`。Flink 推荐使用新的 `Source` API（`org.apache.flink.api.connector.source.Source`），它提供了更好的性能和功能。只有在特定场景下（如需要与旧代码兼容）才建议使用 `SourceFunction`。

## 核心概念

**SourceFunction** 是 Flink 中自定义数据源的**传统接口**（Legacy API）。如果你想从币安 WebSocket 读取数据，可以实现这个接口，但建议优先考虑使用新的 `Source` API。

### 类比理解

这就像：
- **Java 的 Runnable 接口**：定义了 `run()` 方法，实现它就能创建线程
- **Spark 的 Source 接口**：如果你用过 Spark，SourceFunction 在 Flink 中扮演类似角色
- **数据读取器**：定义了"如何读取数据"的规范

### 核心方法

SourceFunction 接口定义了两个必须实现的方法：
1. **`run(SourceContext ctx)`** - 数据生成的核心逻辑
2. **`cancel()`** - 优雅停止数据源

## 源码位置

SourceFunction 接口定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java)

### 接口定义（伪代码）

```java
public interface SourceFunction<T> extends Function, Serializable {
    /**
     * 数据生成的核心方法
     * @param ctx SourceContext，用于发送数据到Flink流中
     */
    void run(SourceContext<T> ctx) throws Exception;

    /**
     * 取消数据源（优雅停止）
     */
    void cancel();

    /**
     * SourceContext：数据发送的上下文
     */
    interface SourceContext<T> {
        // 发送数据（不带时间戳）
        void collect(T element);

        // 发送数据（带时间戳，用于事件时间处理）
        void collectWithTimestamp(T element, long timestamp);

        // 发送Watermark
        void emitWatermark(Watermark mark);

        // 获取检查点锁（用于容错）
        Object getCheckpointLock();
    }
}
```

### 关键理解

1. **SourceFunction 是一个接口**，不是类
2. **必须实现两个方法**：`run()` 和 `cancel()`
3. **`run()` 方法会一直运行**，直到 `cancel()` 被调用
4. **通过 SourceContext 发送数据**到 Flink 流中

## 最小可用例子

```java
// 实现SourceFunction接口
public class BinanceWebSocketSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        // 1. 建立WebSocket连接
        client = new WebSocketClient("wss://stream.binance.com/ws/btcusdt@trade");

        // 2. 设置消息监听器
        client.onMessage(message -> {
            // 3. 解析JSON消息
            Trade trade = parseJson(message);

            // 4. 发送到Flink流中
            synchronized (ctx.getCheckpointLock()) {
                ctx.collect(trade);
            }
        });

        // 5. 保持运行，直到cancel()被调用
        while (isRunning) {
            Thread.sleep(100);
        }
    }

    @Override
    public void cancel() {
        // 停止标志
        isRunning = false;
        // 关闭WebSocket连接
        if (client != null) {
            client.close();
        }
    }
}
```

### 使用方式

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 使用自定义SourceFunction
DataStream<Trade> trades = env.addSource(new BinanceWebSocketSource());

trades.print();
env.execute();
```

## 实现要点

1. **`run()` 方法中的循环**：通常使用 `while (isRunning)` 保持运行
2. **`cancel()` 方法设置标志**：让 `run()` 方法退出循环
3. **使用 `getCheckpointLock()`**：在发送数据时加锁，保证容错一致性
4. **异常处理**：处理网络异常、解析错误等

## 新 API 推荐

Flink 推荐使用新的 `Source` API，它提供了：
- 更好的性能和可扩展性
- 更丰富的功能（如动态分区发现）
- 更好的容错机制

新 API 示例：
```java
import org.apache.flink.api.connector.source.Source;
import org.apache.flink.api.connector.source.SourceReader;
// ... 更多导入

// 实现 Source 接口而不是 SourceFunction
public class BinanceSource implements Source<Trade, ...> {
    // 使用新的 Source API
}
```

## 常见错误

### 错误1：忘记实现 cancel() 方法

```java
// ❌ 错误：只实现run()，没有实现cancel()
public class MySource implements SourceFunction<String> {
    @Override
    public void run(SourceContext<String> ctx) throws Exception {
        while (true) {  // 死循环，无法停止
            ctx.collect("data");
        }
    }
    // 缺少cancel()方法
}

// ✅ 正确：实现两个方法
public class MySource implements SourceFunction<String> {
    private volatile boolean isRunning = true;

    @Override
    public void run(SourceContext<String> ctx) throws Exception {
        while (isRunning) {
            ctx.collect("data");
        }
    }

    @Override
    public void cancel() {
        isRunning = false;  // 让run()退出循环
    }
}
```

### 错误2：不使用 volatile 修饰 isRunning

```java
// ❌ 错误：isRunning不是volatile，可能导致可见性问题
private boolean isRunning = true;  // 错误！

// ✅ 正确：使用volatile保证多线程可见性
private volatile boolean isRunning = true;
```

### 错误3：在 run() 方法中阻塞导致无法响应 cancel()

```java
// ❌ 错误：长时间阻塞，无法及时响应cancel()
@Override
public void run(SourceContext<String> ctx) throws Exception {
    while (isRunning) {
        Thread.sleep(10000);  // 阻塞10秒，cancel()调用后要等10秒才退出
        ctx.collect("data");
    }
}

// ✅ 正确：使用较短的sleep，或检查isRunning
@Override
public void run(SourceContext<String> ctx) throws Exception {
    while (isRunning) {
        ctx.collect("data");
        Thread.sleep(100);  // 短时间阻塞，能及时响应cancel()
    }
}
```

### 错误4：忘记使用 getCheckpointLock()

```java
// ❌ 错误：发送数据时没有加锁
client.onMessage(message -> {
    ctx.collect(data);  // 可能导致检查点不一致
});

// ✅ 正确：使用检查点锁保护
client.onMessage(message -> {
    synchronized (ctx.getCheckpointLock()) {
        ctx.collect(data);  // 保证检查点一致性
    }
});
```

## 什么时候你需要想到这个？

- 当你需要**从自定义数据源读取数据**时（如币安WebSocket）
- 当你需要**实现实时数据采集**时
- 当你看到 Flink 代码中有 `addSource(SourceFunction)` 时
- 当你需要理解 Flink 的**数据源机制**时
- 当你对比 Flink 和 Kafka Consumer 的实现方式时
- ⚠️ **注意**：新项目建议使用新的 `Source` API，而不是 `SourceFunction`


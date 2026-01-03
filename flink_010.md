# SourceFunction的run()方法：数据生成的核心

## 核心概念

**`run(SourceContext ctx)`** 是 SourceFunction 的**核心方法**，所有数据生成逻辑都在这里实现。这个方法会一直运行，直到 `cancel()` 被调用。

### 类比理解

这就像：
- **线程的 run() 方法**：定义了线程要执行的任务
- **Kafka Consumer 的 poll() 循环**：持续从 Kafka 拉取消息
- **事件循环**：持续监听事件并处理

### 核心特点

1. **持续运行**：方法会一直执行，直到被取消
2. **通过 SourceContext 发送数据**：使用 `ctx.collect()` 发送数据
3. **需要处理异常**：网络异常、解析错误等

## 源码位置

SourceFunction 的 run() 方法定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java)

### 方法签名

```java
public interface SourceFunction<T> {
    /**
     * 数据生成的核心方法
     * @param ctx SourceContext，用于发送数据
     */
    void run(SourceContext<T> ctx) throws Exception;
}
```

### 执行流程（伪代码）

```java
public class MySource implements SourceFunction<String> {
    private volatile boolean isRunning = true;

    @Override
    public void run(SourceContext<String> ctx) throws Exception {
        // 1. 初始化（建立连接等）
        initialize();

        // 2. 主循环：持续生成数据
        while (isRunning) {
            // 3. 获取数据（从WebSocket、数据库等）
            String data = fetchData();

            // 4. 发送到Flink流中
            synchronized (ctx.getCheckpointLock()) {
                ctx.collect(data);
            }

            // 5. 控制发送频率（可选）
            Thread.sleep(100);
        }

        // 6. 清理资源
        cleanup();
    }

    @Override
    public void cancel() {
        isRunning = false;  // 让run()方法退出循环
    }
}
```

## 最小可用例子

### 简单示例：生成数字序列

```java
public class NumberSource implements SourceFunction<Long> {
    private volatile boolean isRunning = true;
    private long count = 0;

    @Override
    public void run(SourceContext<Long> ctx) throws Exception {
        while (isRunning && count < 1000) {
            // 发送数据
            synchronized (ctx.getCheckpointLock()) {
                ctx.collect(count);
            }
            count++;
            Thread.sleep(100);  // 每100ms发送一个数字
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }
}
```

### 币安WebSocket示例（简化版）

```java
public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        // 1. 建立WebSocket连接
        client = new WebSocketClient("wss://stream.binance.com/ws/btcusdt@trade");

        // 2. 设置消息处理器
        client.onMessage(message -> {
            try {
                // 3. 解析JSON
                Trade trade = parseJson(message);

                // 4. 发送数据（必须加锁，保证容错一致性）
                synchronized (ctx.getCheckpointLock()) {
                    ctx.collect(trade);
                }
            } catch (Exception e) {
                // 处理解析错误
                logger.error("Failed to parse message", e);
            }
        });

        // 5. 保持运行
        while (isRunning) {
            Thread.sleep(100);
        }
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

## 关键要点

1. **使用 `volatile boolean isRunning`**：让 `cancel()` 可以安全地停止 `run()` 方法
2. **使用 `getCheckpointLock()`**：在发送数据时加锁，保证检查点一致性
3. **异常处理**：捕获并处理可能的异常，避免整个作业失败
4. **资源清理**：在 `cancel()` 或 `run()` 结束时清理资源

## 常见模式

### 模式1：轮询模式

```java
while (isRunning) {
    List<Data> dataList = pollFromSource();  // 轮询获取数据
    for (Data data : dataList) {
        ctx.collect(data);
    }
    Thread.sleep(1000);  // 每秒轮询一次
}
```

### 模式2：事件驱动模式（WebSocket）

```java
source.onEvent(event -> {
    synchronized (ctx.getCheckpointLock()) {
        ctx.collect(event);
    }
});

while (isRunning) {
    Thread.sleep(100);  // 保持线程运行
}
```

## 什么时候你需要想到这个？

- 当你**实现 SourceFunction** 时（核心就是实现这个方法）
- 当你需要**从外部系统读取数据**时（WebSocket、数据库、文件等）
- 当你需要理解 Flink 的**数据生成机制**时
- 当你调试"为什么数据源没有数据"时（检查 run() 方法）
- 当你需要**优化数据源的性能**时（控制发送频率、批处理等）


# SourceFunction的run()方法：数据生成的核心

> ⚠️ **重要提示**：`SourceFunction` 是 Flink 的 **Legacy API**（遗留 API），位于 `legacy` 包下。Flink 推荐使用新的 `Source` API。本文档主要介绍 Legacy API 的实现方式，新项目建议使用新的 `Source` API。

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

> **注意**：此示例需要额外的依赖库（如 WebSocket 客户端和 JSON 解析库）。以下代码仅为概念示例，实际使用时需要添加相应的依赖和实现细节。

```java
// 需要添加的依赖（Maven）：
// <dependency>
//     <groupId>org.java-websocket</groupId>
//     <artifactId>Java-WebSocket</artifactId>
//     <version>1.5.3</version>
// </dependency>
// <dependency>
//     <groupId>com.google.code.gson</groupId>
//     <artifactId>gson</artifactId>
//     <version>2.10.1</version>
// </dependency>

import org.java_websocket.client.WebSocketClient;
import org.java_websocket.handshake.ServerHandshake;
import com.google.gson.Gson;
import java.net.URI;

// Trade 数据类（需要自己定义）
public class Trade {
    public String symbol;
    public double price;
    public double quantity;
    // ... 其他字段
}

public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;
    private final Gson gson = new Gson();

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        // 1. 建立WebSocket连接
        URI uri = new URI("wss://stream.binance.com/ws/btcusdt@trade");
        client = new WebSocketClient(uri) {
            @Override
            public void onMessage(String message) {
                try {
                    // 2. 解析JSON（使用 Gson 或其他 JSON 库）
                    Trade trade = gson.fromJson(message, Trade.class);

                    // 3. 发送数据（必须加锁，保证容错一致性）
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collect(trade);
                    }
                } catch (Exception e) {
                    // 处理解析错误
                    System.err.println("Failed to parse message: " + e.getMessage());
                }
            }

            @Override
            public void onOpen(ServerHandshake handshake) {
                System.out.println("WebSocket connected");
            }

            @Override
            public void onClose(int code, String reason, boolean remote) {
                System.out.println("WebSocket closed: " + reason);
            }

            @Override
            public void onError(Exception ex) {
                System.err.println("WebSocket error: " + ex.getMessage());
            }
        };

        // 4. 连接WebSocket
        client.connect();

        // 5. 保持运行，直到cancel()被调用
        while (isRunning) {
            Thread.sleep(100);
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
        if (client != null) {
            try {
                client.close();
            } catch (Exception e) {
                System.err.println("Error closing WebSocket: " + e.getMessage());
            }
        }
    }
}
```

**使用方式**：
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 使用自定义SourceFunction
DataStream<Trade> trades = env.addSource(new BinanceSource());

trades.print();
env.execute("Binance Trade Stream");
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

## 常见错误

### 错误1：run() 方法没有循环，立即退出

```java
// ❌ 错误：run()方法立即返回，数据源立即停止
@Override
public void run(SourceContext<String> ctx) throws Exception {
    ctx.collect("data1");
    ctx.collect("data2");
    // 方法结束，数据源停止
}

// ✅ 正确：使用循环保持运行
@Override
public void run(SourceContext<String> ctx) throws Exception {
    while (isRunning) {
        ctx.collect("data");
        Thread.sleep(100);
    }
}
```

### 错误2：在 run() 方法中抛出未捕获的异常

```java
// ❌ 错误：异常导致整个作业失败
@Override
public void run(SourceContext<String> ctx) throws Exception {
    while (isRunning) {
        String data = fetchData();  // 可能抛出异常
        ctx.collect(data);  // 如果异常，整个作业失败
    }
}

// ✅ 正确：捕获异常，避免作业失败
@Override
public void run(SourceContext<String> ctx) throws Exception {
    while (isRunning) {
        try {
            String data = fetchData();
            synchronized (ctx.getCheckpointLock()) {
                ctx.collect(data);
            }
        } catch (Exception e) {
            logger.error("Failed to fetch data", e);
            // 继续运行，不抛出异常
        }
    }
}
```

### 错误3：在 run() 方法中创建资源但忘记清理

```java
// ❌ 错误：创建连接但忘记清理
@Override
public void run(SourceContext<String> ctx) throws Exception {
    WebSocketClient client = new WebSocketClient(...);
    client.connect();
    while (isRunning) {
        // 使用client
    }
    // 忘记关闭client，资源泄漏
}

// ✅ 正确：在finally块或cancel()中清理
@Override
public void run(SourceContext<String> ctx) throws Exception {
    WebSocketClient client = null;
    try {
        client = new WebSocketClient(...);
        client.connect();
        while (isRunning) {
            // 使用client
        }
    } finally {
        if (client != null) {
            client.close();
        }
    }
}
```

### 错误4：在 run() 方法中不使用 getCheckpointLock()

```java
// ❌ 错误：多线程环境下发送数据没有加锁
client.onMessage(message -> {
    ctx.collect(data);  // 可能导致检查点不一致
});

// ✅ 正确：使用检查点锁
client.onMessage(message -> {
    synchronized (ctx.getCheckpointLock()) {
        ctx.collect(data);
    }
});
```

### 错误5：run() 方法中的循环条件错误

```java
// ❌ 错误：循环条件错误，无法退出
@Override
public void run(SourceContext<String> ctx) throws Exception {
    while (true) {  // 死循环，即使cancel()被调用也无法退出
        ctx.collect("data");
    }
}

// ✅ 正确：检查isRunning标志
@Override
public void run(SourceContext<String> ctx) throws Exception {
    while (isRunning) {  // 可以通过cancel()设置isRunning=false退出
        ctx.collect("data");
    }
}
```

## 什么时候你需要想到这个？

- 当你**实现 SourceFunction** 时（核心就是实现这个方法）
- 当你需要**从外部系统读取数据**时（WebSocket、数据库、文件等）
- 当你需要理解 Flink 的**数据生成机制**时
- 当你调试"为什么数据源没有数据"时（检查 run() 方法）
- 当你需要**优化数据源的性能**时（控制发送频率、批处理等）


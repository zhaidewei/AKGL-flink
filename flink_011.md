# SourceFunction的cancel()方法：优雅停止

> ⚠️ **重要提示**：`SourceFunction` 是 Flink 的 **Legacy API**（遗留 API）。Flink 推荐使用新的 `Source` API。本文档主要介绍 Legacy API 的实现方式。

## 核心概念

**`cancel()`** 方法用于**优雅停止**数据源。当 Flink 需要停止数据源时（比如作业取消、故障恢复），会调用这个方法。

### 类比理解

这就像：
- **线程的 interrupt()**：通知线程停止
- **资源的 close()**：关闭连接、释放资源
- **信号处理**：接收停止信号，执行清理工作

### 核心职责

1. **设置停止标志**：让 `run()` 方法退出循环
2. **关闭连接**：关闭 WebSocket、数据库连接等
3. **释放资源**：清理占用的资源

## 源码位置

SourceFunction 的 cancel() 方法定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/functions/source/legacy/SourceFunction.java)

### 方法签名

```java
public interface SourceFunction<T> {
    /**
     * 取消数据源（优雅停止）
     * 实现应该设置标志，让run()方法退出循环
     */
    void cancel();
}
```

### 实现模式（伪代码）

```java
public class MySource implements SourceFunction<String> {
    // 使用volatile保证可见性
    private volatile boolean isRunning = true;
    private WebSocketClient client;

    @Override
    public void run(SourceContext<String> ctx) throws Exception {
        client = connect();

        // 检查isRunning标志
        while (isRunning) {
            // 处理数据
            String data = client.receive();
            ctx.collect(data);
        }

        // run()方法退出，数据源停止
    }

    @Override
    public void cancel() {
        // 1. 设置停止标志（让run()退出循环）
        isRunning = false;

        // 2. 关闭连接
        if (client != null) {
            client.close();
        }

        // 3. 释放其他资源
        cleanup();
    }
}
```

## 最小可用例子

### 币安WebSocket示例

```java
public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private WebSocketClient client;

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        client = new WebSocketClient("wss://stream.binance.com/ws/btcusdt@trade");
        client.onMessage(msg -> {
            Trade trade = parseJson(msg);
            synchronized (ctx.getCheckpointLock()) {
                ctx.collect(trade);
            }
        });

        // 检查isRunning，如果为false则退出
        while (isRunning) {
            Thread.sleep(100);
        }
    }

    @Override
    public void cancel() {
        // 设置标志，让run()退出while循环
        isRunning = false;

        // 关闭WebSocket连接
        if (client != null) {
            try {
                client.close();
            } catch (Exception e) {
                logger.error("Error closing WebSocket", e);
            }
        }
    }
}
```

## 关键要点

1. **使用 `volatile boolean`**：保证多线程可见性
2. **在 `run()` 中检查标志**：`while (isRunning)` 让循环可以退出
3. **关闭连接**：避免资源泄漏
4. **异常处理**：关闭连接时可能抛出异常，需要捕获

## 常见错误

### 错误1：忘记设置标志

```java
@Override
public void cancel() {
    client.close();  // 只关闭连接，但run()中的循环还在运行！
}
```

**正确做法**：
```java
@Override
public void cancel() {
    isRunning = false;  // 必须先设置标志
    client.close();
}
```

### 错误2：不使用 volatile

```java
private boolean isRunning = true;  // 错误：不是volatile
```

**正确做法**：
```java
private volatile boolean isRunning = true;  // 正确：使用volatile
```

## 什么时候你需要想到这个？

- 当你**实现 SourceFunction** 时（必须实现这个方法）
- 当你需要**优雅关闭数据源**时（避免资源泄漏）
- 当你调试"数据源无法停止"的问题时（检查 cancel() 实现）
- 当你需要理解 Flink 的**作业生命周期**时
- 当你需要**处理作业取消和故障恢复**时


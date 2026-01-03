# 重连间隔策略：指数退避算法

## 核心概念

**指数退避（Exponential Backoff）** 是一种重连策略：每次重连失败后，等待时间逐渐增加（指数增长），避免频繁重连对服务器造成压力。

### 算法原理

```
第1次重连：等待 1秒
第2次重连：等待 2秒
第3次重连：等待 4秒
第4次重连：等待 8秒
...
最大等待时间：60秒（避免等待时间过长）
```

### 公式

```java
delay = min(1000 * (2 ^ retryCount), maxDelay)
```

## 最小可用例子

```java
public class BinanceSource implements SourceFunction<Trade> {
    private int retryCount = 0;
    private static final long MAX_DELAY = 60000;  // 最大延迟60秒

    private void scheduleReconnect(SourceContext<Trade> ctx) {
        retryCount++;

        // 指数退避：2^retryCount 秒，但不超过MAX_DELAY
        long delay = Math.min(1000 * (1L << retryCount), MAX_DELAY);

        logger.info("Reconnecting in " + delay + "ms (attempt " + retryCount + ")");

        new Thread(() -> {
            try {
                Thread.sleep(delay);
                if (isRunning) {
                    connect(ctx);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }

    // 连接成功后重置计数
    private void onMessage(String message) {
        retryCount = 0;  // 连接成功，重置重连计数
        // 处理消息...
    }
}
```

## 延迟时间表

| 重连次数 | 延迟时间 | 累计等待时间 |
|---------|---------|-------------|
| 1 | 1秒 | 1秒 |
| 2 | 2秒 | 3秒 |
| 3 | 4秒 | 7秒 |
| 4 | 8秒 | 15秒 |
| 5 | 16秒 | 31秒 |
| 6 | 32秒 | 63秒 |
| 7+ | 60秒（最大） | ... |

## 完整示例

```java
public class BinanceSource implements SourceFunction<Trade> {
    private volatile boolean isRunning = true;
    private int retryCount = 0;
    private static final int MAX_RETRIES = 10;
    private static final long MAX_DELAY = 60000;

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        connect(ctx);

        while (isRunning) {
            Thread.sleep(100);
        }
    }

    private void connect(SourceContext<Trade> ctx) {
        WebSocketClient client = new WebSocketClient(...) {
            @Override
            public void onMessage(String message) {
                // 连接成功，重置计数
                retryCount = 0;
                // 处理消息...
            }

            @Override
            public void onClose(int code, String reason, boolean remote) {
                if (isRunning && retryCount < MAX_RETRIES) {
                    scheduleReconnect(ctx);
                }
            }
        };

        client.connect();
    }

    private void scheduleReconnect(SourceContext<Trade> ctx) {
        retryCount++;

        // 指数退避
        long delay = Math.min(1000 * (1L << retryCount), MAX_DELAY);

        logger.info("Reconnecting in " + delay + "ms (attempt " + retryCount + "/" + MAX_RETRIES + ")");

        new Thread(() -> {
            try {
                Thread.sleep(delay);
                if (isRunning) {
                    connect(ctx);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }
}
```

## 为什么使用指数退避？

1. **减少服务器压力**：避免频繁重连
2. **给网络恢复时间**：网络问题可能需要时间恢复
3. **节省资源**：减少无效的重连尝试
4. **行业标准**：大多数系统都使用这种策略

## 其他重连策略

### 固定间隔

```java
long delay = 5000;  // 固定5秒
```

### 线性增长

```java
long delay = 1000 * retryCount;  // 1秒、2秒、3秒...
```

## 什么时候你需要想到这个？

- 当你需要**实现 WebSocket 重连**时（避免频繁重连）
- 当你需要**优化数据源性能**时（减少无效重连）
- 当你需要**处理网络波动**时（给网络恢复时间）
- 当你需要**遵循最佳实践**时（指数退避是标准做法）
- 当你需要**减少服务器压力**时（避免DDoS式重连）


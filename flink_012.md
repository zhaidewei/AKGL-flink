# SourceReaderContext：读取器的上下文对象

> ✅ **重要提示**：`SourceReaderContext` 是新 Source API 中提供给 SourceReader 的上下文对象。它替代了 Legacy API 中的 `SourceContext`。

## 核心概念

**SourceReaderContext** 是新 Source API 中提供给 SourceReader 的**上下文对象**。它提供了访问指标、配置、发送事件等方法。

### 类比理解

这就像：
- **线程的 Context**：提供线程相关的信息和方法
- **服务的 Context**：提供服务相关的配置和功能
- **执行环境**：提供执行相关的上下文信息

### 核心方法

SourceReaderContext 提供了以下关键方法：
1. **`metricGroup()`** - 获取指标组
2. **`getConfiguration()`** - 获取配置
3. **`sendSplitRequest()`** - 发送分片请求
4. **`sendSourceEventToCoordinator()`** - 发送事件到协调器
5. **`getIndexOfSubtask()`** - 获取子任务索引

## 源码位置

SourceReaderContext 接口定义在：
[flink-core/src/main/java/org/apache/flink/api/connector/source/SourceReaderContext.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/connector/source/SourceReaderContext.java)

### 接口定义（关键方法）

```java
public interface SourceReaderContext {
    /**
     * 获取指标组
     */
    SourceReaderMetricGroup metricGroup();

    /**
     * 获取配置
     */
    Configuration getConfiguration();

    /**
     * 获取本地主机名
     */
    String getLocalHostName();

    /**
     * 获取子任务索引
     */
    int getIndexOfSubtask();

    /**
     * 发送分片请求到 SplitEnumerator
     */
    void sendSplitRequest();

    /**
     * 发送事件到协调器
     */
    void sendSourceEventToCoordinator(SourceEvent sourceEvent);

    /**
     * 获取用户代码类加载器
     */
    UserCodeClassLoader getUserCodeClassLoader();
}
```

## 最小可用例子

```java
public class BinanceWebSocketReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private final SourceReaderContext context;

    public BinanceWebSocketReader(SourceReaderContext context, String symbol) {
        this.context = context;
    }

    @Override
    public void start() {
        // 1. 获取配置
        Configuration config = context.getConfiguration();
        String symbol = config.getString("binance.symbol", "btcusdt");

        // 2. 获取子任务索引
        int subtaskIndex = context.getIndexOfSubtask();
        logger.info("Starting reader for subtask {}", subtaskIndex);

        // 3. 获取指标组（用于监控）
        SourceReaderMetricGroup metrics = context.metricGroup();
        Counter recordCounter = metrics.getIOMetricGroup().getNumRecordsInCounter();

        // 4. 建立 WebSocket 连接
        // ...
    }

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        // 使用 output 发送数据（不是 context）
        // ...
    }

    private void handleError(Exception e) {
        // 发送错误事件到协调器
        context.sendSourceEventToCoordinator(new ErrorSourceEvent(e.getMessage()));
    }
}
```

## 关键方法详解

### 1. metricGroup() - 获取指标组

```java
SourceReaderMetricGroup metrics = context.metricGroup();
Counter recordCounter = metrics.getIOMetricGroup().getNumRecordsInCounter();
recordCounter.inc();  // 增加记录计数
```

### 2. sendSplitRequest() - 请求分片

```java
// 当需要更多分片时，发送请求
context.sendSplitRequest();
```

### 3. sendSourceEventToCoordinator() - 发送事件

```java
// 发送自定义事件到协调器
context.sendSourceEventToCoordinator(new CustomSourceEvent(data));
```

## 与 Legacy SourceContext 的对比

| 特性 | Legacy SourceContext | 新 SourceReaderContext |
|------|---------------------|----------------------|
| 数据发送 | collect() | 不提供（使用 ReaderOutput） |
| 时间戳 | collectWithTimestamp() | 不提供（使用 ReaderOutput） |
| 指标 | 不支持 | 支持（metricGroup()） |
| 配置 | 不支持 | 支持（getConfiguration()） |
| 事件发送 | 不支持 | 支持（sendSourceEventToCoordinator()） |
| 推荐度 | ⚠️ 不推荐 | ✅ 推荐 |

## 常见错误

### 错误1：在 SourceReaderContext 中发送数据

```java
// ❌ 错误：SourceReaderContext 不提供数据发送方法
context.collect(trade);  // 不存在！

// ✅ 正确：使用 ReaderOutput 发送数据
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
    output.collect(trade);  // 使用 output
}
```

### 错误2：忘记使用 context 获取配置

```java
// ❌ 错误：硬编码配置
String symbol = "btcusdt";

// ✅ 正确：从 context 获取配置
Configuration config = context.getConfiguration();
String symbol = config.getString("binance.symbol", "btcusdt");
```

## 什么时候你需要想到这个？

- 当你**实现 SourceReader** 时（需要访问上下文信息）
- 当你需要**获取配置**时（从 context 获取）
- 当你需要**发送事件**时（发送到协调器）
- 当你需要**访问指标**时（监控数据源性能）
- 当你需要**获取子任务信息**时（并行度、索引等）

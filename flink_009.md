# Source接口：自定义数据源的基础（新API）

> ✅ **重要提示**：`Source` 接口是 Flink 的**新 API**（推荐使用），位于 `org.apache.flink.api.connector.source` 包下。它提供了比 Legacy `SourceFunction` 更好的性能、可扩展性和容错机制。

## 核心概念

**Source** 接口是 Flink 中自定义数据源的**新 API 接口**。如果你想从币安 WebSocket 读取数据，应该实现这个接口，而不是使用 Legacy 的 `SourceFunction`。

### 类比理解

这就像：
- **工厂模式**：Source 是一个工厂，负责创建 SplitEnumerator 和 SourceReader
- **策略模式**：定义了数据源的策略（如何读取、如何分片）
- **数据读取器工厂**：定义了"如何创建数据读取器"的规范

### 核心组件

Source 接口涉及三个主要组件：
1. **`Source<T, SplitT, EnumChkT>`** - 主接口，定义数据源
2. **`SplitEnumerator<SplitT, EnumChkT>`** - 分片枚举器，管理数据分片（对于 WebSocket 可能不需要）
3. **`SourceReader<T, SplitT>`** - 数据读取器，实际读取数据

## 源码位置

Source 接口定义在：
[flink-core/src/main/java/org/apache/flink/api/connector/source/Source.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/connector/source/Source.java)

### 接口定义（伪代码）

```java
public interface Source<T, SplitT extends SourceSplit, EnumChkT>
        extends SourceReaderFactory<T, SplitT> {

    /**
     * 获取数据源的有界性（有界流还是无界流）
     */
    Boundedness getBoundedness();

    /**
     * 创建新的 SplitEnumerator（分片枚举器）
     * 对于 WebSocket 这种单流数据源，可能返回简单的实现
     */
    SplitEnumerator<SplitT, EnumChkT> createEnumerator(
            SplitEnumeratorContext<SplitT> enumContext) throws Exception;

    /**
     * 从检查点恢复 SplitEnumerator
     */
    SplitEnumerator<SplitT, EnumChkT> restoreEnumerator(
            SplitEnumeratorContext<SplitT> enumContext, EnumChkT checkpoint) throws Exception;

    /**
     * 创建分片的序列化器
     */
    SimpleVersionedSerializer<SplitT> getSplitSerializer();

    /**
     * 创建枚举器检查点的序列化器
     */
    SimpleVersionedSerializer<EnumChkT> getEnumeratorCheckpointSerializer();
}
```

### 关键理解

1. **Source 是一个接口**，不是类
2. **需要实现多个方法**：创建枚举器、创建读取器、序列化器等
3. **支持有界和无界流**：通过 `getBoundedness()` 指定
4. **支持分片**：通过 SplitEnumerator 管理数据分片（对于 WebSocket 可能不需要）

## 最小可用例子

### 币安 WebSocket Source（简化版）

```java
import org.apache.flink.api.connector.source.*;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.core.io.SimpleVersionedSerializer;

// 定义分片类型（对于 WebSocket，可能只需要一个分片）
public class BinanceWebSocketSplit implements SourceSplit {
    private final String splitId;

    public BinanceWebSocketSplit(String splitId) {
        this.splitId = splitId;
    }

    @Override
    public String splitId() {
        return splitId;
    }
}

// 实现 Source 接口
public class BinanceWebSocketSource implements Source<Trade, BinanceWebSocketSplit, Void> {
    private final String symbol;  // 交易对，如 "btcusdt"

    public BinanceWebSocketSource(String symbol) {
        this.symbol = symbol;
    }

    @Override
    public Boundedness getBoundedness() {
        return Boundedness.CONTINUOUS_UNBOUNDED;  // 无界流
    }

    @Override
    public SourceReader<Trade, BinanceWebSocketSplit> createReader(
            SourceReaderContext readerContext) {
        return new BinanceWebSocketReader(readerContext, symbol);
    }

    @Override
    public SplitEnumerator<BinanceWebSocketSplit, Void> createEnumerator(
            SplitEnumeratorContext<BinanceWebSocketSplit> enumContext) {
        // 对于 WebSocket，创建一个简单的分片
        return new BinanceWebSocketSplitEnumerator(enumContext);
    }

    @Override
    public SplitEnumerator<BinanceWebSocketSplit, Void> restoreEnumerator(
            SplitEnumeratorContext<BinanceWebSocketSplit> enumContext, Void checkpoint) {
        return new BinanceWebSocketSplitEnumerator(enumContext);
    }

    @Override
    public SimpleVersionedSerializer<BinanceWebSocketSplit> getSplitSerializer() {
        return new BinanceWebSocketSplitSerializer();
    }

    @Override
    public SimpleVersionedSerializer<Void> getEnumeratorCheckpointSerializer() {
        return new VoidSerializer();
    }
}

// 以下辅助类需要实现：

// 1. BinanceWebSocketSplitEnumerator: 管理数据分片
// 对于 WebSocket 这种单流数据源，可以实现一个简单的枚举器
class BinanceWebSocketSplitEnumerator implements SplitEnumerator<BinanceWebSocketSplit, Void> {
    private final SplitEnumeratorContext<BinanceWebSocketSplit> context;

    public BinanceWebSocketSplitEnumerator(SplitEnumeratorContext<BinanceWebSocketSplit> context) {
        this.context = context;
    }

    @Override
    public void start() {
        // 对于 WebSocket，创建一个分片并分配给 reader
        BinanceWebSocketSplit split = new BinanceWebSocketSplit("split-0");
        context.assignSplit(split, 0);  // 分配给第一个 reader
    }

    // ... 其他必需的方法实现
}

// 2. BinanceWebSocketSplitSerializer: 序列化分片
class BinanceWebSocketSplitSerializer implements SimpleVersionedSerializer<BinanceWebSocketSplit> {
    @Override
    public int getVersion() {
        return 1;
    }

    @Override
    public byte[] serialize(BinanceWebSocketSplit split) {
        // 序列化 splitId
        return split.splitId().getBytes();
    }

    @Override
    public BinanceWebSocketSplit deserialize(int version, byte[] serialized) {
        // 反序列化
        return new BinanceWebSocketSplit(new String(serialized));
    }
}

// 3. VoidSerializer: 序列化 Void 类型（用于检查点）
class VoidSerializer implements SimpleVersionedSerializer<Void> {
    @Override
    public int getVersion() {
        return 1;
    }

    @Override
    public byte[] serialize(Void obj) {
        return new byte[0];
    }

    @Override
    public Void deserialize(int version, byte[] serialized) {
        return null;
    }

    @Override
    public TypeInformation<Trade> getProducedType() {
        return TypeInformation.of(Trade.class);
    }
}
```

### 使用方式

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 使用新的 Source API
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Trade Source"
);

trades.print();
env.execute();
```

## 实现要点

1. **实现 `getBoundedness()`**：指定是有界流还是无界流
2. **实现 `createReader()`**：创建 SourceReader 实例
3. **实现 `createEnumerator()`**：创建 SplitEnumerator（对于简单场景可以返回简单实现）
4. **实现序列化器**：为分片和检查点提供序列化器

## 与 Legacy SourceFunction 的对比

| 特性 | Legacy SourceFunction | 新 Source API |
|------|----------------------|---------------|
| 接口复杂度 | 简单（2个方法） | 较复杂（多个方法） |
| 性能 | 一般 | 更好 |
| 可扩展性 | 有限 | 更好 |
| 容错机制 | 基础 | 更完善 |
| 分片支持 | 不支持 | 支持 |
| 推荐度 | ⚠️ 不推荐 | ✅ 推荐 |

## 常见错误

### 错误1：忘记实现 getBoundedness()

```java
// ❌ 错误：没有实现 getBoundedness()
public class MySource implements Source<String, MySplit, Void> {
    // 缺少 getBoundedness() 方法
}

// ✅ 正确：实现所有必需的方法
public class MySource implements Source<String, MySplit, Void> {
    @Override
    public Boundedness getBoundedness() {
        return Boundedness.CONTINUOUS_UNBOUNDED;  // 无界流
    }
    // ... 其他方法
}
```

### 错误2：返回 null 的序列化器

```java
// ❌ 错误：返回 null
@Override
public SimpleVersionedSerializer<MySplit> getSplitSerializer() {
    return null;  // 会导致运行时错误
}

// ✅ 正确：返回有效的序列化器
@Override
public SimpleVersionedSerializer<MySplit> getSplitSerializer() {
    return new MySplitSerializer();
}
```

### 错误3：在 createReader() 中创建多个实例

```java
// ❌ 错误：每次调用都创建新实例（可能导致资源泄漏）
@Override
public SourceReader<Trade, BinanceWebSocketSplit> createReader(
        SourceReaderContext readerContext) {
    return new BinanceWebSocketReader(readerContext, symbol);
    // 注意：这是正确的，但需要确保每个 reader 独立管理资源
}
```

## 什么时候你需要想到这个？

- 当你需要**实现自定义数据源**时（如币安 WebSocket）
- 当你需要**使用 Flink 新 API**时（推荐使用新 API）
- 当你需要**更好的性能和可扩展性**时（新 API 提供更好的性能）
- 当你需要**支持数据分片**时（新 API 支持分片）
- 当你需要**更好的容错机制**时（新 API 提供更完善的容错）

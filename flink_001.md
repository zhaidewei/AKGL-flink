# DataStream：Flink中的流数据抽象

## 核心概念

**DataStream** 是 Flink 中表示**无界数据流**的核心抽象。如果你熟悉 Spark 的 DStream 或 Kafka 的流概念，DataStream 在 Flink 中扮演类似的角色。

### 类比理解

想象一下：
- **DataStream** 就像一条**永不停止的传送带**
- 传送带上的每个物品就是一个**数据元素**
- 你可以对传送带上的物品进行各种操作（转换、过滤、分组等）
- 但传送带本身一直在流动，不会停下来

### 与 Spark 的对比

如果你用过 Spark Streaming：
- Spark 的 DStream 是**微批处理**（每几秒处理一批数据）
- Flink 的 DataStream 是**真正的流处理**（数据到达即处理，延迟更低）

## 源码位置

DataStream 的类定义在：

[flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java)

### 核心结构（伪代码）

```java
public class DataStream<T> {
    // 每个DataStream都关联一个执行环境
    protected final StreamExecutionEnvironment environment;

    // 每个DataStream都包含一个转换（Transformation）
    protected final Transformation<T> transformation;

    // DataStream的构造函数
    public DataStream(StreamExecutionEnvironment environment,
                      Transformation<T> transformation) {
        this.environment = environment;
        this.transformation = transformation;
    }

    // DataStream提供了很多转换方法（这就是API）
    public <R> DataStream<R> map(MapFunction<T, R> mapper) { ... }
    public DataStream<T> filter(FilterFunction<T> filter) { ... }
    // ... 更多方法
}
```

### 关键理解

1. **DataStream 是一个类**，不是接口
2. **泛型 `<T>`** 表示流中元素的类型（比如 `DataStream<String>` 表示字符串流）
3. **每个 DataStream 都关联一个 Transformation**，它记录了"这个流是怎么来的"
4. **DataStream 提供了很多方法**（map、filter、keyBy等），这些方法的集合就是 DataStream API

## 最小可用例子

```java
// 1. 创建执行环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 2. 创建一个DataStream（从集合创建，用于测试）
DataStream<String> words = env.fromElements("hello", "world", "flink");

// 3. 对DataStream进行转换
DataStream<String> upperWords = words.map(word -> word.toUpperCase());

// 4. 输出结果
upperWords.print();

// 5. 执行作业
env.execute("My Flink Job");
```

在这个例子中：
- `words` 是一个 `DataStream<String>`，包含3个字符串
- `upperWords` 是另一个 `DataStream<String>`，是转换后的结果
- 每个转换操作都返回**新的 DataStream**，原来的不变

## 什么时候你需要想到这个？

- 当你需要处理**持续不断的数据流**（如币安WebSocket交易数据）时
- 当你看到 Flink 代码中有 `DataStream<...>` 类型声明时
- 当你需要理解 Flink 的**流式处理模型**时
- 当你对比 Flink 和 Spark Streaming 的区别时
- 当你开始设计一个实时数据处理系统时


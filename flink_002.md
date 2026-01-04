# DataStream的不可变性：为什么每次转换都产生新流

## 核心概念

**DataStream 是不可变的**。这意味着：每次对 DataStream 进行转换操作（如 map、filter），都会产生一个**新的 DataStream 对象**，原来的 DataStream 保持不变。

### 类比理解

这就像：
- **字符串的不可变性**：`String s = "hello"; String upper = s.toUpperCase();` - `s` 本身不变，`toUpperCase()` 返回新字符串
- **Spark RDD 的不可变性**：如果你用过 Spark，RDD 也是不可变的，每次转换都产生新的 RDD

### 为什么这样设计？

1. **安全性**：避免意外修改原始数据流
2. **可追溯性**：可以保留原始流和转换后的流
3. **优化空间**：Flink 可以优化整个转换链，而不是单个操作

## 源码位置

DataStream 的 map 方法在：

[flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java)

### 核心机制（概念性伪代码）

> **注意**：以下为简化版概念性伪代码，用于说明不可变性的原理。实际实现通过 `map()` → `transform()` → `doTransform()` 的调用链完成，并且返回的是 `SingleOutputStreamOperator<R>`（DataStream 的子类）。

```java
public class DataStream<T> {
    // 原始的transformation（不可变）
    protected final Transformation<T> transformation;
    protected final StreamExecutionEnvironment environment;

    // map方法：返回新的DataStream，不修改当前DataStream
    // 实际返回类型是 SingleOutputStreamOperator<R>
    public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper) {
        // 1. 提取输出类型信息
        TypeInformation<R> outType = TypeExtractor.getMapReturnTypes(
            clean(mapper),
            getType(),
            Utils.getCallLocationName(),
            true
        );

        // 2. 调用 transform 方法（间接调用链）
        return transform("Map", outType, new StreamMap<>(clean(mapper)));
    }

    // transform 方法（实际实现）
    protected <R> SingleOutputStreamOperator<R> transform(
            String operatorName,
            TypeInformation<R> outTypeInfo,
            OneInputStreamOperator<T, R> operator) {

        // 3. 创建 OneInputTransformation
        // 注意：实际使用 StreamOperatorFactory，这里简化表示
        OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
            this.transformation,              // 父转换（当前流的转换）
            operatorName,                     // 操作名称（如 "Map"）
            SimpleOperatorFactory.of(operator), // 算子工厂（包装算子）
            outTypeInfo,                      // 输出类型
            environment.getParallelism(),     // 并行度
            false                              // parallelismConfigured标志
        );

        // 4. 创建新的 DataStream（实际是 SingleOutputStreamOperator）
        SingleOutputStreamOperator<R> returnStream =
            new SingleOutputStreamOperator<>(environment, resultTransform);

        // 5. 将转换添加到环境（用于后续构建执行图）
        environment.addOperator(resultTransform);

        // 6. 返回新流（原来的DataStream完全不变）
        return returnStream;
    }
}
```

**关键理解**：
1. `map()` 方法不直接创建 `OneInputTransformation`，而是通过 `transform()` → `doTransform()` 间接完成
2. 实际返回类型是 `SingleOutputStreamOperator<R>`，它是 `DataStream<R>` 的子类
3. 创建 Transformation 时需要提供并行度等参数
4. 新的 Transformation 会被添加到环境的 `transformations` 列表中

### 关键理解

1. **每次转换都创建新的 Transformation 对象**
2. **新的 Transformation 记录"父转换"是谁**（形成转换链）
3. **返回新的 DataStream，原来的 DataStream 不变**
4. **Flink 在 execute() 时才会真正执行这些转换**

## 最小可用例子

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 原始流
DataStream<String> original = env.fromElements("hello", "world");

// 转换1：转大写（产生新流，original不变）
DataStream<String> upper = original.map(s -> s.toUpperCase());

// 转换2：过滤（产生新流，upper不变）
DataStream<String> filtered = upper.filter(s -> s.length() > 4);

// 此时有三个DataStream对象：
// - original: 包含 "hello", "world"
// - upper: 包含 "HELLO", "WORLD"
// - filtered: 包含 "HELLO", "WORLD"（都满足长度>4）

// 可以同时使用原始流和转换后的流
original.print();  // 输出: hello, world
filtered.print();  // 输出: HELLO, WORLD
```

## 转换链的构建

当你写：
```java
stream.map(...).filter(...).keyBy(...)
```

实际上构建了一个转换链：
```
原始Transformation → MapTransformation → FilterTransformation → KeyByTransformation
```

每个转换都记录"我的父转换是谁"，Flink 在 execute() 时根据这个链构建执行图。

## 什么时候你需要想到这个？

- 当你对同一个 DataStream 进行多次转换时
- 当你疑惑"为什么我的转换没有生效"时（可能用错了流对象）
- 当你需要**复用原始流**进行不同转换时
- 当你理解 Flink 的**懒加载机制**时（转换只是记录，不立即执行）
- 当你对比 Flink 和 Spark 的转换模型时


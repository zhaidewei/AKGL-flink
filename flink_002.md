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

### 核心机制（伪代码）

```java
public class DataStream<T> {
    // 原始的transformation（不可变）
    protected final Transformation<T> transformation;

    // map方法：返回新的DataStream，不修改当前DataStream
    public <R> DataStream<R> map(MapFunction<T, R> mapper) {
        // 1. 创建一个新的Transformation
        OneInputTransformation<T, R> transform = new OneInputTransformation<>(
            this.transformation,  // 父转换（当前流的转换）
            "Map",                // 操作名称
            new StreamMap<>(clean(mapper)),  // 实际的map算子
            getType(),            // 输入类型
            TypeExtractor.getMapReturnTypes(...)  // 输出类型
        );

        // 2. 基于新Transformation创建新的DataStream
        return new DataStream<>(this.environment, transform);
        // 注意：原来的DataStream（this）完全没有被修改
    }
}
```

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


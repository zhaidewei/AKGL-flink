# 实现序列化：确保数据可被Flink序列化

## 核心概念

Flink 需要在集群中传输数据，因此所有数据类必须**可序列化**。实现 `Serializable` 接口是最基本的要求。

### 核心要求

1. **实现Serializable**：类必须实现 `java.io.Serializable`
2. **无参构造函数**：Flink序列化需要
3. **字段可序列化**：所有字段也必须是可序列化的

## 最小可用例子

```java
import java.io.Serializable;

public class Trade implements Serializable {
    private static final long serialVersionUID = 1L;  // 可选，但推荐

    private String symbol;
    private double price;
    private double quantity;
    private long tradeTime;

    // 无参构造函数（必须）
    public Trade() {}

    // Getter和Setter
    public String getSymbol() { return symbol; }
    public void setSymbol(String symbol) { this.symbol = symbol; }

    // ... 其他getter/setter
}
```

## 币安交易数据示例

### 完整的可序列化类

```java
import java.io.Serializable;

public class Trade implements Serializable {
    private static final long serialVersionUID = 1L;

    private String symbol;
    private double price;
    private double quantity;
    private long tradeTime;
    private boolean isBuyerMaker;

    // 无参构造函数（Flink序列化必需）
    public Trade() {}

    // 带参构造函数
    public Trade(String symbol, double price, double quantity,
                 long tradeTime, boolean isBuyerMaker) {
        this.symbol = symbol;
        this.price = price;
        this.quantity = quantity;
        this.tradeTime = tradeTime;
        this.isBuyerMaker = isBuyerMaker;
    }

    // Getter和Setter（Flink需要）
    public String getSymbol() { return symbol; }
    public void setSymbol(String symbol) { this.symbol = symbol; }

    public double getPrice() { return price; }
    public void setPrice(double price) { this.price = price; }

    public double getQuantity() { return quantity; }
    public void setQuantity(double quantity) { this.quantity = quantity; }

    public long getTradeTime() { return tradeTime; }
    public void setTradeTime(long tradeTime) { this.tradeTime = tradeTime; }

    public boolean isBuyerMaker() { return isBuyerMaker; }
    public void setBuyerMaker(boolean buyerMaker) { isBuyerMaker = buyerMaker; }
}
```

## 序列化要求

### 1. 实现Serializable

```java
public class Trade implements Serializable {
    // ...
}
```

### 2. 无参构造函数

```java
// 必须有无参构造函数
public Trade() {}
```

### 3. 字段可序列化

```java
// ✅ 可序列化的类型
private String symbol;        // String可序列化
private double price;         // 基本类型可序列化
private long tradeTime;       // 基本类型可序列化

// ❌ 不可序列化的类型（需要特殊处理）
private WebSocketClient client;  // 不能序列化，不应该放在数据类中
```

## 使用Flink的序列化器（高级）

```java
// 如果默认序列化有问题，可以使用Flink的序列化器
env.getConfig().registerTypeWithKryoSerializer(Trade.class, TradeSerializer.class);
```

## 常见错误

### 错误1：忘记实现Serializable

```java
public class Trade {  // ❌ 缺少implements Serializable
    // ...
}
```

### 错误2：没有无参构造函数

```java
public class Trade implements Serializable {
    public Trade(String symbol) {  // ❌ 只有带参构造函数
        this.symbol = symbol;
    }
}
```

### 错误3：字段不可序列化

```java
public class Trade implements Serializable {
    private WebSocketClient client;  // ❌ 不可序列化
}
```

## 什么时候你需要想到这个？

- 当你需要**设计数据类**时（必须可序列化）
- 当你遇到**序列化错误**时（检查是否实现Serializable）
- 当你需要**理解Flink的数据传输**时（序列化是基础）
- 当你需要**调试序列化问题**时（检查字段类型）
- 当你需要**优化序列化性能**时（使用Flink序列化器）



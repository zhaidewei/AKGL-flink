# 币安Trade数据模型：设计交易数据类

> ⚠️ **重要提示**：本文档中的示例代码使用 `SourceFunction`（Legacy API）实现。Flink 推荐使用新的 `Source` API。本文档主要介绍 Legacy API 的数据模型设计。

## 核心概念

设计币安交易数据的 Java 类，用于表示从币安 WebSocket 接收到的交易数据。

### 核心字段

根据币安 WebSocket API，Trade 数据包含：
- `symbol` - 交易对（如"BTCUSDT"）
- `price` - 价格
- `quantity` - 数量
- `tradeTime` - 交易时间（事件时间）
- `isBuyerMaker` - 是否为买方主动

## 最小可用例子

```java
import java.io.Serializable;

public class Trade implements Serializable {
    private String symbol;        // 交易对，如 "BTCUSDT"
    private double price;          // 价格
    private double quantity;       // 数量
    private long tradeTime;        // 交易时间（事件时间，毫秒）
    private boolean isBuyerMaker;  // 是否为买方主动

    // 无参构造函数（Flink序列化需要）
    public Trade() {}

    // 构造函数
    public Trade(String symbol, double price, double quantity, long tradeTime, boolean isBuyerMaker) {
        this.symbol = symbol;
        this.price = price;
        this.quantity = quantity;
        this.tradeTime = tradeTime;
        this.isBuyerMaker = isBuyerMaker;
    }

    // Getter和Setter
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

    @Override
    public String toString() {
        return "Trade{symbol='" + symbol + "', price=" + price +
               ", quantity=" + quantity + ", time=" + tradeTime + "}";
    }
}
```

## 币安JSON解析

### JSON格式

```json
{
  "e": "trade",
  "E": 123456789,
  "s": "BTCUSDT",
  "t": 12345,
  "p": "0.001",
  "q": "100",
  "b": 88,
  "a": 50,
  "T": 123456785,
  "m": true,
  "M": true
}
```

### 解析方法

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

public class TradeParser {
    private static final ObjectMapper objectMapper = new ObjectMapper();

    public static Trade parse(String json) throws Exception {
        JsonNode node = objectMapper.readTree(json);

        Trade trade = new Trade();
        trade.setSymbol(node.get("s").asText());              // "s": 交易对
        trade.setPrice(node.get("p").asDouble());            // "p": 价格
        trade.setQuantity(node.get("q").asDouble());          // "q": 数量
        trade.setTradeTime(node.get("T").asLong());           // "T": 交易时间
        trade.setBuyerMaker(node.get("m").asBoolean());      // "m": 是否为买方主动

        return trade;
    }
}
```

## 在SourceFunction中使用

```java
public class BinanceSource implements SourceFunction<Trade> {
    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        WebSocketClient client = new WebSocketClient(...) {
            @Override
            public void onMessage(String message) {
                try {
                    // 解析JSON为Trade对象
                    Trade trade = TradeParser.parse(message);

                    // 发送到Flink流
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(trade, trade.getTradeTime());
                    }
                } catch (Exception e) {
                    logger.error("Failed to parse trade", e);
                }
            }
        };

        client.connect();
        while (isRunning) {
            Thread.sleep(100);
        }
    }
}
```

## 关键要点

1. **实现Serializable**：Flink需要序列化
2. **无参构造函数**：Flink序列化需要
3. **Getter/Setter**：Flink需要访问字段
4. **事件时间字段**：tradeTime用于事件时间处理

## 什么时候你需要想到这个？

- 当你需要**设计数据模型**时（Trade类的设计）
- 当你需要**解析币安JSON数据**时（字段映射）
- 当你需要**实现序列化**时（Flink要求）
- 当你需要**提取事件时间**时（tradeTime字段）
- 当你需要**理解数据流**时（从JSON到对象）



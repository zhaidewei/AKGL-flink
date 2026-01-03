# JSON解析：将币安消息转换为Java对象

## 核心概念

币安 WebSocket 返回的是 JSON 格式的字符串，需要解析为 Java 对象（如 Trade）才能发送到 Flink 流中。

### 常用JSON库

1. **Jackson** - 功能强大，性能好（推荐）
2. **Gson** - Google 的 JSON 库，简单易用
3. **Fastjson** - 阿里巴巴的 JSON 库（不推荐，有安全问题）

## 使用 Jackson（推荐）

### Maven 依赖

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.2</version>
</dependency>
```

### 解析示例

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.JsonNode;

public class BinanceSource implements SourceFunction<Trade> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    private Trade parseJson(String json) throws Exception {
        JsonNode node = objectMapper.readTree(json);

        Trade trade = new Trade();
        trade.setSymbol(node.get("s").asText());              // "BTCUSDT"
        trade.setPrice(node.get("p").asDouble());            // 价格
        trade.setQuantity(node.get("q").asDouble());          // 数量
        trade.setTradeTime(node.get("T").asLong());           // 交易时间
        trade.setIsBuyerMaker(node.get("m").asBoolean());     // 是否为买方主动

        return trade;
    }
}
```

### 直接映射到对象

```java
// Trade类需要有无参构造函数和getter/setter
public class Trade {
    @JsonProperty("s")
    private String symbol;

    @JsonProperty("p")
    private double price;

    @JsonProperty("q")
    private double quantity;

    @JsonProperty("T")
    private long tradeTime;

    @JsonProperty("m")
    private boolean isBuyerMaker;

    // getter/setter...
}

// 解析
private Trade parseJson(String json) throws Exception {
    return objectMapper.readValue(json, Trade.class);
}
```

## 使用 Gson

### Maven 依赖

```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.10.1</version>
</dependency>
```

### 解析示例

```java
import com.google.gson.Gson;
import com.google.gson.JsonObject;

public class BinanceSource implements SourceFunction<Trade> {
    private final Gson gson = new Gson();

    private Trade parseJson(String json) {
        JsonObject obj = gson.fromJson(json, JsonObject.class);

        Trade trade = new Trade();
        trade.setSymbol(obj.get("s").getAsString());
        trade.setPrice(obj.get("p").getAsDouble());
        trade.setQuantity(obj.get("q").getAsDouble());
        trade.setTradeTime(obj.get("T").getAsLong());
        trade.setIsBuyerMaker(obj.get("m").getAsBoolean());

        return trade;
    }
}
```

## 币安 Trade JSON 格式

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

## 完整示例

```java
public class BinanceSource implements SourceFunction<Trade> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void run(SourceContext<Trade> ctx) throws Exception {
        WebSocketClient client = new WebSocketClient(new URI("wss://stream.binance.com:9443/ws/btcusdt@trade")) {
            @Override
            public void onMessage(String message) {
                try {
                    // 解析JSON
                    Trade trade = parseJson(message);

                    // 发送到Flink流
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(trade, trade.getTradeTime());
                    }
                } catch (Exception e) {
                    logger.error("Failed to parse message: " + message, e);
                }
            }
        };

        client.connect();

        while (isRunning) {
            Thread.sleep(100);
        }
    }

    private Trade parseJson(String json) throws Exception {
        JsonNode node = objectMapper.readTree(json);

        Trade trade = new Trade();
        trade.setSymbol(node.get("s").asText());
        trade.setPrice(node.get("p").asDouble());
        trade.setQuantity(node.get("q").asDouble());
        trade.setTradeTime(node.get("T").asLong());
        trade.setIsBuyerMaker(node.get("m").asBoolean());

        return trade;
    }
}
```

## 性能优化

1. **重用 ObjectMapper**：不要每次解析都创建新实例
2. **异常处理**：解析失败不要影响整个作业
3. **字段选择**：只解析需要的字段

## 什么时候你需要想到这个？

- 当你需要**解析币安返回的 JSON 数据**时（WebSocket消息是JSON格式）
- 当你需要**将 JSON 转换为 Java 对象**时（发送到Flink流）
- 当你需要**选择 JSON 解析库**时（Jackson vs Gson）
- 当你需要**优化数据源性能**时（JSON解析是CPU密集型）
- 当你需要**处理实时数据流**时（快速解析很重要）


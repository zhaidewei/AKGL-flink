# 币安Ticker数据模型：设计行情数据类

## 核心概念

设计币安 Ticker（24小时行情）数据的 Java 类，用于表示币安 WebSocket 返回的行情数据。

### 核心字段

根据币安 WebSocket API，Ticker 数据包含：
- `symbol` - 交易对
- `lastPrice` - 最新价格
- `priceChange` - 24小时价格变化
- `priceChangePercent` - 24小时价格变化百分比
- `volume` - 24小时成交量
- `timestamp` - 时间戳

## 最小可用例子

```java
import java.io.Serializable;

public class Ticker implements Serializable {
    private String symbol;              // 交易对
    private double lastPrice;           // 最新价格
    private double priceChange;         // 24小时价格变化
    private double priceChangePercent;  // 24小时价格变化百分比
    private double volume;              // 24小时成交量
    private long timestamp;             // 时间戳（事件时间）

    // 无参构造函数
    public Ticker() {}

    // Getter和Setter
    public String getSymbol() { return symbol; }
    public void setSymbol(String symbol) { this.symbol = symbol; }

    public double getLastPrice() { return lastPrice; }
    public void setLastPrice(double lastPrice) { this.lastPrice = lastPrice; }

    public double getPriceChange() { return priceChange; }
    public void setPriceChange(double priceChange) { this.priceChange = priceChange; }

    public double getPriceChangePercent() { return priceChangePercent; }
    public void setPriceChangePercent(double priceChangePercent) {
        this.priceChangePercent = priceChangePercent;
    }

    public double getVolume() { return volume; }
    public void setVolume(double volume) { this.volume = volume; }

    public long getTimestamp() { return timestamp; }
    public void setTimestamp(long timestamp) { this.timestamp = timestamp; }

    @Override
    public String toString() {
        return "Ticker{symbol='" + symbol + "', price=" + lastPrice +
               ", change=" + priceChangePercent + "%}";
    }
}
```

## 币安JSON解析

### JSON格式（简化）

```json
{
  "e": "24hrTicker",
  "E": 123456789,
  "s": "BTCUSDT",
  "c": "50000.00",
  "P": "2.5",
  "p": "1250.00",
  "v": "1000.00",
  "E": 123456789
}
```

### 解析方法

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

public class TickerParser {
    private static final ObjectMapper objectMapper = new ObjectMapper();

    public static Ticker parse(String json) throws Exception {
        JsonNode node = objectMapper.readTree(json);

        Ticker ticker = new Ticker();
        ticker.setSymbol(node.get("s").asText());                    // "s": 交易对
        ticker.setLastPrice(node.get("c").asDouble());              // "c": 最新价格
        ticker.setPriceChange(node.get("p").asDouble());            // "p": 价格变化
        ticker.setPriceChangePercent(node.get("P").asDouble());     // "P": 价格变化百分比
        ticker.setVolume(node.get("v").asDouble());                 // "v": 成交量
        ticker.setTimestamp(node.get("E").asLong());                // "E": 时间戳

        return ticker;
    }
}
```

## 在SourceFunction中使用

```java
public class BinanceTickerSource implements SourceFunction<Ticker> {
    @Override
    public void run(SourceContext<Ticker> ctx) throws Exception {
        WebSocketClient client = new WebSocketClient(
            new URI("wss://stream.binance.com:9443/ws/btcusdt@ticker")) {

            @Override
            public void onMessage(String message) {
                try {
                    Ticker ticker = TickerParser.parse(message);
                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collectWithTimestamp(ticker, ticker.getTimestamp());
                    }
                } catch (Exception e) {
                    logger.error("Failed to parse ticker", e);
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

## 与Trade的区别

| 特性 | Trade | Ticker |
|------|-------|--------|
| 数据频率 | 每条交易 | 定期更新（如每秒） |
| 数据内容 | 单笔交易 | 24小时统计 |
| 使用场景 | 实时交易流 | 行情监控 |

## 什么时候你需要想到这个？

- 当你需要**设计Ticker数据模型**时（行情数据类）
- 当你需要**解析币安Ticker JSON**时（字段映射）
- 当你需要**监控行情变化**时（价格、涨跌幅等）
- 当你需要**理解不同数据流**时（Trade vs Ticker）
- 当你需要**实现行情数据源**时（Ticker SourceFunction）



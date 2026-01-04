# 币安WebSocket API：理解连接URL和订阅格式

## 核心概念

币安提供了 WebSocket Streams API，用于实时接收交易数据。理解 URL 格式和订阅方式是实现数据源的基础。

## 币安WebSocket URL格式

### 单个流（Single Stream）

```
wss://stream.binance.com:9443/ws/<stream>
```

**示例**：
- `wss://stream.binance.com:9443/ws/btcusdt@trade` - BTC/USDT 交易流
- `wss://stream.binance.com:9443/ws/ethusdt@ticker` - ETH/USDT 行情流

### 组合流（Combined Streams）

```
wss://stream.binance.com:9443/stream?streams=<stream1>/<stream2>/<stream3>
```

**示例**：
```
wss://stream.binance.com:9443/stream?streams=btcusdt@trade/ethusdt@trade
```

## 常用流类型

### 1. Trade Stream（交易流）

**格式**：`<symbol>@trade`

**示例**：
```
wss://stream.binance.com:9443/ws/btcusdt@trade
```

**返回数据格式**（JSON）：
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

**字段说明**：
- `e`: 事件类型（"trade"）
- `E`: 事件时间（毫秒）
- `s`: 交易对（"BTCUSDT"）
- `p`: 价格
- `q`: 数量
- `T`: 交易时间（事件时间，毫秒）
- `m`: 是否为买方主动

### 2. Ticker Stream（24小时行情流）

**格式**：`<symbol>@ticker`

**示例**：
```
wss://stream.binance.com:9443/ws/btcusdt@ticker
```

**返回数据**：包含24小时统计信息（价格、涨跌幅、成交量等）

### 3. Kline Stream（K线流）

**格式**：`<symbol>@kline_<interval>`

**示例**：
```
wss://stream.binance.com:9443/ws/btcusdt@kline_1m  // 1分钟K线
wss://stream.binance.com:9443/ws/btcusdt@kline_5m  // 5分钟K线
```

## 最小可用例子

### 连接单个交易流

```java
import org.apache.flink.api.connector.source.*;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class BinanceTradeReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    private static final String WS_URL = "wss://stream.binance.com:9443/ws/btcusdt@trade";
    private WebSocketClient client;
    private final BlockingQueue<Trade> recordQueue = new LinkedBlockingQueue<>();
    private volatile boolean isRunning = true;

    @Override
    public void start() {
        try {
            client = new WebSocketClient(new URI(WS_URL)) {
                @Override
                public void onMessage(String message) {
                    // 解析JSON
                    Trade trade = parseTradeJson(message);
                    // 放入队列（非阻塞）
                    recordQueue.offer(trade);
                }

                @Override
                public void onError(Exception ex) {
                    logger.error("WebSocket error", ex);
                }
            };

            client.connect();
        } catch (Exception e) {
            throw new RuntimeException("Failed to connect WebSocket", e);
        }
    }

    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        Trade trade = recordQueue.poll();
        if (trade != null) {
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }
        return isRunning ? InputStatus.NOTHING_AVAILABLE : InputStatus.END_OF_INPUT;
    }

    @Override
    public void close() throws Exception {
        isRunning = false;
        if (client != null) {
            client.close();
        }
    }

    // ... 其他必需的方法
}
```

### 连接多个流（组合流）

```java
// 同时订阅BTC和ETH的交易流
String streams = "btcusdt@trade/ethusdt@trade";
String url = "wss://stream.binance.com:9443/stream?streams=" + streams;

@Override
public void start() {
    try {
        client = new WebSocketClient(new URI(url)) {
            @Override
            public void onMessage(String message) {
                // 消息格式：{"stream":"btcusdt@trade","data":{...}}
                // 需要解析stream字段判断是哪个流
                Trade trade = parseCombinedStream(message);
                recordQueue.offer(trade);
            }
        };
        client.connect();
    } catch (Exception e) {
        throw new RuntimeException("Failed to connect WebSocket", e);
    }
}
```

## 数据解析示例

### Trade JSON 解析

```java
public Trade parseTradeJson(String json) {
    JSONObject obj = new JSONObject(json);

    Trade trade = new Trade();
    trade.setSymbol(obj.getString("s"));           // "BTCUSDT"
    trade.setPrice(obj.getDouble("p"));           // 价格
    trade.setQuantity(obj.getDouble("q"));        // 数量
    trade.setTradeTime(obj.getLong("T"));        // 交易时间（事件时间）
    trade.setIsBuyerMaker(obj.getBoolean("m"));   // 是否为买方主动

    return trade;
}
```

## 注意事项

1. **URL 格式**：注意大小写（`btcusdt` 全小写）
2. **端口**：9443（WSS）或 443
3. **连接限制**：单个IP有连接数限制
4. **重连**：需要实现自动重连机制
5. **心跳**：币安会定期发送ping，需要响应pong

## 什么时候你需要想到这个？

- 当你需要**连接币安 WebSocket**时（第一步就是理解URL格式）
- 当你需要**订阅不同的数据流**时（trade、ticker、kline等）
- 当你需要**解析币安返回的JSON数据**时
- 当你需要**实现币安数据源**时（SourceFunction）
- 当你需要**处理多个交易对**时（组合流）


# MariaDB R2DBC Exchange Leak Issue Reproduction

Minimal repository to reproduce the Exchange leak issue in MariaDB R2DBC driver.

## Problem Overview

This repository demonstrates the issue where Exchange objects remain in the exchangeQueue when API clients are aborted.
When the leak occurs, the exchange's hasDemand is false and isCancelled is true at the sendCommand stage, preventing onRequest from being called. This keeps demand at 0, causing messages to accumulate in the receiverQueue without being emitted to the exchange via onNext, and the Exchange at the head of the queue remains unpolled, resulting in a leak.

## Environment

- Spring Boot 3.5.6
- Spring WebFlux
- Spring Data R2DBC
- MariaDB R2DBC Connector 1.3.0
- Java 21

## Setup

### 1. Add Debug Logs to SimpleClient.java

To debug the Exchange leak issue, you need to add debug logs to the MariaDB R2DBC driver's `SimpleClient.java` file.

1. Download the `SimpleClient.java` file from the MariaDB R2DBC connector repository (version 1.3.0):
   ```bash
   curl -o src/main/java/org/mariadb/r2dbc/client/SimpleClient.java \
     https://raw.githubusercontent.com/mariadb-corporation/mariadb-connector-r2dbc/1.3.0/src/main/java/org/mariadb/r2dbc/client/SimpleClient.java
   ```

2. Apply the following debug log patches:

   **In `sendCommand` method (around line 630):**
   ```diff
   @@ -627,6 +627,7 @@
              if (message instanceof PreparePacket) {
                decoder.addPrepare(((PreparePacket) message).getSql());
              }
   +          logger.info("sendCommand ==> exchange:{}, hasDemand: {}, cancelled: {}", exchange, exchange.hasDemand(), exchange.isCancelled());
              sink.onRequest(value -> messageSubscriber.onRequest(exchange, value));
              this.requestSink.emitNext(message, Sinks.EmitFailureHandler.FAIL_FAST);
            } else {
   ```

   **In `sendCommand` (prepare+execute) method (around line 683):**
   ```diff
   @@ -680,6 +681,7 @@
                new Exchange(
                    sink, DecoderState.PREPARE_AND_EXECUTE_RESPONSE, preparePacket.getSql());
            if (this.exchangeQueue.offer(exchange)) {
   +          logger.info("sendCommand (prepare+execute) ==> exchange:{}, hasDemand: {}, cancelled: {}", exchange, exchange.hasDemand(), exchange.isCancelled());
              sink.onRequest(value -> messageSubscriber.onRequest(exchange, value));
              decoder.addPrepare(preparePacket.getSql());
              this.requestSink.emitNext(preparePacket, Sinks.EmitFailureHandler.FAIL_FAST);
   ```

   **In `onNext` method of `ServerMessageSubscriber` inner class (around line 766):**
   ```diff
   @@ -764,6 +766,11 @@

          this.receiverDemands.decrementAndGet();
          Exchange exchange = this.exchangeQueue.peek();
   +
   +      logger.info("onNext ==> message: {}, exchange: {}, hasDemand: {}, cancelled: {}, receiverQueue.size: {}",
   +          message.getClass().getSimpleName(), exchange,
   +          exchange != null ? exchange.hasDemand() : "null",
   +          exchange != null ? exchange.isCancelled() : "null",
   +          this.receiverQueue.size());

          // nothing buffered => directly emit message
          ReferenceCountUtil.retain(message);
   ```

   **In `onRequest` method of `ServerMessageSubscriber` inner class (around line 789):**
   ```diff
   @@ -787,6 +794,8 @@
        }

        public void onRequest(Exchange exchange, long n) {
   +      logger.info("onRequest ==> exchange: {}, demand: {}, hasDemand: {}, cancelled: {}",
   +          exchange, n, exchange.hasDemand(), exchange.isCancelled());
          exchange.incrementDemand(n);
          requestQueueFilling();
          tryDrainQueue();
   ```

### 2. Start MariaDB

```bash
docker compose up -d
```

### 3. Start the application

```bash
./gradlew bootRun
```

## How to Reproduce the Exchange Leak

![Exchange leak demonstration](demo.gif)

### 1. Trigger client aborts with a short timeout

First, execute 100 requests with a short timeout to force a client aborts. And then, send a normal request.

```bash
for i in $(seq 100); do curl "http://localhost:8080/api/users" -m 0.002; done

curl "http://localhost:8080/api/users"
```

Verify that this request hangs indefinitely (as shown in the demonstration above).

**Note**: Depending on your environment, you may need to repeat the above steps several times to reproduce the leak.

## Log Analysis

When the leak occurs, you can verify the issue by analyzing the debug logs:

### 1. Identify the leaked Exchange

Search for an Exchange with `hasDemand: false` and `cancelled: true`:

```bash
$ grep 'hasDemand: false, cancelled: true' app.log | head -n 1
2025-10-08T11:17:50.776+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : sendCommand ==> exchange:org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true
```

### 2. Trace the Exchange behavior

Extract all logs related to the leaked Exchange (e.g., `Exchange@3bdc39f7`):

```bash
$ grep 'Exchange@3bdc39f7' app.log
```

**Key observations:**

1. The Exchange is created with `hasDemand: false, cancelled: true` in the `sendCommand` method
2. **No `onRequest` calls are made** for this cancelled Exchange, leaving demand at 0
3. Messages continue to arrive via `onNext`, but cannot be emitted due to zero demand
4. The `receiverQueue` size keeps growing: 0 → 1 → 2 → ... → 187

**Example log output:**

```
2025-10-08T11:17:50.776+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : sendCommand ==> exchange:org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true
2025-10-08T11:17:50.776+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : onNext ==> message: ColumnCountPacket, exchange: org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true, receiverQueue.size: 0
2025-10-08T11:17:50.777+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : onNext ==> message: ColumnDefinitionPacket, exchange: org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true, receiverQueue.size: 1
2025-10-08T11:17:50.777+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : onNext ==> message: ColumnDefinitionPacket, exchange: org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true, receiverQueue.size: 2
2025-10-08T11:17:50.777+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : onNext ==> message: ColumnDefinitionPacket, exchange: org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true, receiverQueue.size: 3
2025-10-08T11:17:50.777+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : onNext ==> message: EofPacket, exchange: org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true, receiverQueue.size: 4
2025-10-08T11:17:50.777+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : onNext ==> message: RowPacket, exchange: org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true, receiverQueue.size: 5
...
2025-10-08T11:17:51.933+09:00  INFO 94276 --- [r2dbc-exchange-leak-issue] [actor-tcp-nio-3] org.mariadb.r2dbc.client.SimpleClient    : onNext ==> message: ColumnDefinitionPacket, exchange: org.mariadb.r2dbc.client.Exchange@3bdc39f7, hasDemand: false, cancelled: true, receiverQueue.size: 187
```

This demonstrates that:
- The cancelled Exchange remains at the head of the `exchangeQueue`
- Messages accumulate in the `receiverQueue` without being consumed
- The Exchange is never removed from the queue, blocking all later requests

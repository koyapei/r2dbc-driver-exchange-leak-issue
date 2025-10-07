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

2. Add debug log in the `sendCommand` method (around line 630):
   ```java
   logger.info("sendCommand ==> exchange:{}, hasDemand: {}, cancelled: {}",
       exchange, exchange.hasDemand(), exchange.isCancelled());
   ```

3. Add debug log in the `onNext` method of `ServerMessageSubscriber` inner class:
   ```java
   logger.info("onNext ==> message: {}, exchange: {}, hasDemand: {}, cancelled: {}, receiverQueue.size: {}",
       message.getClass().getSimpleName(), exchange,
       exchange != null ? exchange.hasDemand() : "null",
       exchange != null ? exchange.isCancelled() : "null",
       this.receiverQueue.size());
   ```

4. Add debug log in the `onRequest` method of `ServerMessageSubscriber` inner class:
   ```java
   logger.info("onRequest ==> exchange: {}, demand: {}, hasDemand: {}, cancelled: {}",
       exchange, n, exchange.hasDemand(), exchange.isCancelled());
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

### 1. Trigger client aborts with short timeout

First, execute 100 requests with short timeout to force client aborts. And then, send a normal request.

```bash
for i in $(seq 100); do curl "http://localhost:8080/api/users" -m 0.002; done

curl "http://localhost:8080/api/users"
```

Verify that this request hangs.

**Note**: Depending on your environment, you may need to repeat the above steps several times to reproduce the leak.

# Race condition in RpcServer connectionHandler allows exceeding maxActiveConnections

## Description
There’s a race condition in the `hildr::RpcServer.java` class within the `connectionHandler` method. The method first [checks the current number of active connections](https://github.com/optimism-java/hildr/blob/280b82c27cffc323570a5586f848b3bf0a25a08b/src/main/java/io/optimism/rpc/RpcServer.java#L112) using `activeConnectionsCount.get()`:
```java
            if (activeConnectionsCount.get() >= maxActiveConnections) {
                // disallow new connections to prevent DoS
```

and then, depending on the result, either [increments the count](https://github.com/optimism-java/hildr/blob/280b82c27cffc323570a5586f848b3bf0a25a08b/src/main/java/io/optimism/rpc/RpcServer.java#L123) or [rejects the connection](https://github.com/optimism-java/hildr/blob/280b82c27cffc323570a5586f848b3bf0a25a08b/src/main/java/io/optimism/rpc/RpcServer.java#L114-L118). However, because these two steps "checking and incrementing" are handled separately, there’s a small window where multiple threads can see the same count and both decide it’s safe to proceed.
This race condition allows multiple connections to be accepted even if only one should have been, potentially leading to a higher-than-expected number of active connections.

### Example Scenario
- `maxActiveConnections` is set to 80, and `activeConnectionsCount` is currently at 79.
- Two threads, Thread A and Thread B, attempt to establish connections simultaneously.
- Both threads check `activeConnectionsCount.get()` and see a value of 79, deciding they are within the limit.
- Both threads proceed to `incrementAndGet()`, resulting in a final count of 81, which exceeds `maxActiveConnections`.

The example above describes two simultaneous threads, but there can be situations where more threads encounter this condition at the same time and initiate connections even after the cap is reached, which can be harmful to the node.

## Recommendation
To resolve this issue, make the connection count check and increment operation atomic by using `incrementAndGet()` directly in the if condition. This change ensures that each thread’s connection increment and check are handled in a single, atomic step, thereby eliminating the race condition.

```diff
    private Handler<HttpConnection> connectionHandler() {
        return connection -> {
-           if (activeConnectionsCount.get() >= maxActiveConnections) {
+           if (activeConnectionsCount.incrementAndGet() > maxActiveConnections) {
                // disallow new connections to prevent DoS
                logger.warn(
                        "Rejecting new connection from {}. Max {} active connections limit reached.",
                        connection.remoteAddress(),
-                       activeConnectionsCount.getAndIncrement());
+                       activeConnectionsCount.get());
+               activeConnectionsCount.decrementAndGet();
                connection.close();
            } else {
                logger.debug(
                        "Opened connection from {}. Total of active connections: {}/{}",
                        connection.remoteAddress(),
-                       activeConnectionsCount.incrementAndGet(),
+                       activeConnectionsCount.get(),
                        maxActiveConnections);
            }
            connection.closeHandler(c -> logger.debug(
                    "Connection closed from {}. Total of active connections: {}/{}",
                    connection.remoteAddress(),
                    activeConnectionsCount.decrementAndGet(),
                    maxActiveConnections));
        };
    }
```

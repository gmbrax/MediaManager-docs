# Client Development Guide

This guide walks you through building a complete client application for MediaManager Core. You'll learn how to connect, send commands, receive responses, and handle errors - everything needed to create a functional MediaManager client.

---

## Prerequisites

Before starting, make sure you have:

1. **Development environment set up** - See [[Development Environment Setup]]
2. **Understanding of Protocol Buffers** - See [[Protocol Buffers Guide]]
3. **Understanding of Unix Sockets** - See [[Unix Socket Communication]]
4. **MediaManager Core running** - The daemon must be active for testing

---

## Overview

Building a MediaManager client involves:

1. **Connecting** to the Unix socket
2. **Creating commands** using Protocol Buffers
3. **Wrapping commands** in Request messages
4. **Sending requests** with length-prefix framing
5. **Receiving responses** and parsing results
6. **Handling errors** gracefully
7. **Managing connection lifecycle**

---

## Project Setup

### Java Client Setup

Create a new Maven project:
```bash
mkdir mediamanager-client
cd mediamanager-client
```

Create `pom.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>mediamanager-client</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <protobuf.version>3.24.0</protobuf.version>
    </properties>

    <dependencies>
        <!-- Protocol Buffers -->
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>${protobuf.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Protobuf Maven Plugin -->
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>
                        com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}
                    </protocArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- OS Maven Plugin (for protobuf) -->
            <plugin>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.7.1</version>
                <executions>
                    <execution>
                        <phase>initialize</phase>
                        <goals>
                            <goal>detect</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### Copy Protocol Buffer Files

Copy the `.proto` files from MediaManager Core:
```bash
# Create proto directory
mkdir -p src/main/proto

# Copy proto files from Core repository
cp /path/to/MediaManager-core/src/main/proto/*.proto src/main/proto/
```

Your project structure should look like:
```
mediamanager-client/
├── pom.xml
└── src/
    └── main/
        ├── java/
        │   └── com/example/client/
        └── proto/
            ├── transport.proto
            └── test.proto
```

### Generate Java Classes
```bash
# Generate protobuf classes
mvn clean generate-sources

# Verify generated files
ls target/generated-sources/protobuf/java/com/mediamanager/protocol/
# Should show: TransportProtocol.java, TestProtocol.java
```

---

## Step 1: Create Connection Manager

Create `src/main/java/com/example/client/ConnectionManager.java`:
```java
package com.example.client;

import java.io.IOException;
import java.net.StandardProtocolFamily;
import java.net.UnixDomainSocketAddress;
import java.nio.channels.SocketChannel;
import java.nio.file.Path;

public class ConnectionManager {
    
    private SocketChannel channel;
    private final Path socketPath;
    private boolean connected;
    
    public ConnectionManager(String socketPath) {
        this.socketPath = Path.of(socketPath);
        this.connected = false;
    }
    
    /**
     * Connect to MediaManager Core
     */
    public void connect() throws IOException {
        if (connected) {
            throw new IllegalStateException("Already connected");
        }
        
        UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketPath);
        channel = SocketChannel.open(StandardProtocolFamily.UNIX);
        channel.connect(address);
        
        connected = true;
        System.out.println("Connected to MediaManager Core at: " + socketPath);
    }
    
    /**
     * Disconnect from MediaManager Core
     */
    public void disconnect() throws IOException {
        if (!connected) {
            return;
        }
        
        if (channel != null && channel.isOpen()) {
            channel.close();
        }
        
        connected = false;
        System.out.println("Disconnected from MediaManager Core");
    }
    
    /**
     * Check if connected
     */
    public boolean isConnected() {
        return connected && channel != null && channel.isOpen();
    }
    
    /**
     * Get the socket channel
     */
    public SocketChannel getChannel() {
        if (!connected) {
            throw new IllegalStateException("Not connected");
        }
        return channel;
    }
}
```

---

## Step 2: Create Protocol Handler

Create `src/main/java/com/example/client/ProtocolHandler.java`:
```java
package com.example.client;

import com.google.protobuf.ByteString;
import com.mediamanager.protocol.TransportProtocol;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.util.UUID;

public class ProtocolHandler {
    
    private static final int MAX_MESSAGE_SIZE = 1024 * 1024; // 1MB
    
    /**
     * Send a request to the server
     */
    public void sendRequest(SocketChannel channel, byte[] commandPayload, String commandType) 
            throws IOException {
        
        // Build Request message
        TransportProtocol.Request request = TransportProtocol.Request.newBuilder()
            .setRequestId(UUID.randomUUID().toString())
            .setPayload(ByteString.copyFrom(commandPayload))
            .putHeaders("Command-Type", commandType)
            .build();
        
        // Serialize to bytes
        byte[] requestBytes = request.toByteArray();
        int messageSize = requestBytes.length;
        
        System.out.println("Sending request " + request.getRequestId() + 
                         " (" + messageSize + " bytes)");
        
        // Create buffer with length-prefix
        ByteBuffer buffer = ByteBuffer.allocate(4 + messageSize);
        buffer.putInt(messageSize);      // 4-byte size header
        buffer.put(requestBytes);        // Protobuf payload
        buffer.flip();
        
        // Write to socket
        while (buffer.hasRemaining()) {
            channel.write(buffer);
        }
    }
    
    /**
     * Receive a response from the server
     */
    public TransportProtocol.Response receiveResponse(SocketChannel channel) 
            throws IOException {
        
        // Read 4-byte size header
        ByteBuffer sizeBuffer = ByteBuffer.allocate(4);
        int bytesRead = 0;
        
        while (bytesRead < 4) {
            int read = channel.read(sizeBuffer);
            if (read == -1) {
                throw new IOException("Connection closed by server");
            }
            bytesRead += read;
        }
        
        sizeBuffer.flip();
        int messageSize = sizeBuffer.getInt();
        
        // Validate message size
        if (messageSize <= 0 || messageSize > MAX_MESSAGE_SIZE) {
            throw new IOException("Invalid message size: " + messageSize);
        }
        
        System.out.println("Receiving response (" + messageSize + " bytes)");
        
        // Read payload
        ByteBuffer messageBuffer = ByteBuffer.allocate(messageSize);
        bytesRead = 0;
        
        while (bytesRead < messageSize) {
            int read = channel.read(messageBuffer);
            if (read == -1) {
                throw new IOException("Connection closed while reading message");
            }
            bytesRead += read;
        }
        
        messageBuffer.flip();
        
        // Deserialize protobuf
        byte[] messageBytes = new byte[messageSize];
        messageBuffer.get(messageBytes);
        
        TransportProtocol.Response response = TransportProtocol.Response.parseFrom(messageBytes);
        
        System.out.println("Received response " + response.getRequestId() + 
                         " (status: " + response.getStatusCode() + ")");
        
        return response;
    }
    
    /**
     * Helper: Send request and receive response in one call
     */
    public TransportProtocol.Response sendAndReceive(
            SocketChannel channel, 
            byte[] commandPayload, 
            String commandType) throws IOException {
        
        sendRequest(channel, commandPayload, commandType);
        return receiveResponse(channel);
    }
}
```

---

## Step 3: Create MediaManager Client

Create `src/main/java/com/example/client/MediaManagerClient.java`:
```java
package com.example.client;

import com.mediamanager.protocol.TransportProtocol;
import com.mediamanager.protocol.TestProtocol;

import java.io.IOException;

public class MediaManagerClient {
    
    private final ConnectionManager connectionManager;
    private final ProtocolHandler protocolHandler;
    
    public MediaManagerClient(String socketPath) {
        this.connectionManager = new ConnectionManager(socketPath);
        this.protocolHandler = new ProtocolHandler();
    }
    
    /**
     * Connect to MediaManager Core
     */
    public void connect() throws IOException {
        connectionManager.connect();
    }
    
    /**
     * Disconnect from MediaManager Core
     */
    public void disconnect() throws IOException {
        connectionManager.disconnect();
    }
    
    /**
     * Check if connected
     */
    public boolean isConnected() {
        return connectionManager.isConnected();
    }
    
    /**
     * Send an Echo command
     */
    public TestProtocol.EchoResponse echo(String message) throws IOException {
        // Create Echo command
        TestProtocol.EchoCommand command = TestProtocol.EchoCommand.newBuilder()
            .setMessage(message)
            .build();
        
        // Send and receive
        TransportProtocol.Response response = protocolHandler.sendAndReceive(
            connectionManager.getChannel(),
            command.toByteArray(),
            "Echo"
        );
        
        // Check status
        if (response.getStatusCode() != 200) {
            String errorMsg = response.getHeadersOrDefault("Error-Message", "Unknown error");
            throw new IOException("Server error: " + errorMsg);
        }
        
        // Parse response payload
        return TestProtocol.EchoResponse.parseFrom(response.getPayload());
    }
    
    /**
     * Send a Heartbeat command
     */
    public TestProtocol.HeartbeatResponse heartbeat() throws IOException {
        // Create Heartbeat command
        TestProtocol.HeartbeatCommand command = TestProtocol.HeartbeatCommand.newBuilder()
            .setClientTimestamp(System.currentTimeMillis())
            .build();
        
        // Send and receive
        TransportProtocol.Response response = protocolHandler.sendAndReceive(
            connectionManager.getChannel(),
            command.toByteArray(),
            "Heartbeat"
        );
        
        // Check status
        if (response.getStatusCode() != 200) {
            String errorMsg = response.getHeadersOrDefault("Error-Message", "Unknown error");
            throw new IOException("Server error: " + errorMsg);
        }
        
        // Parse response payload
        return TestProtocol.HeartbeatResponse.parseFrom(response.getPayload());
    }
    
    /**
     * Send a Close command
     */
    public TestProtocol.CloseResponse close() throws IOException {
        // Create Close command
        TestProtocol.CloseCommand command = TestProtocol.CloseCommand.newBuilder()
            .build();
        
        // Send and receive
        TransportProtocol.Response response = protocolHandler.sendAndReceive(
            connectionManager.getChannel(),
            command.toByteArray(),
            "Close"
        );
        
        // Check status
        if (response.getStatusCode() != 200) {
            String errorMsg = response.getHeadersOrDefault("Error-Message", "Unknown error");
            throw new IOException("Server error: " + errorMsg);
        }
        
        // Parse response payload
        return TestProtocol.CloseResponse.parseFrom(response.getPayload());
    }
}
```

---

## Step 4: Create Main Application

Create `src/main/java/com/example/client/Main.java`:
```java
package com.example.client;

import com.mediamanager.protocol.TestProtocol;

public class Main {
    
    public static void main(String[] args) {
        // Default socket path
        String socketPath = "/tmp/mediamanager.sock";
        
        // Override with command-line argument if provided
        if (args.length > 0) {
            socketPath = args[0];
        }
        
        MediaManagerClient client = new MediaManagerClient(socketPath);
        
        try {
            // Connect to MediaManager Core
            System.out.println("Connecting to MediaManager Core...");
            client.connect();
            System.out.println();
            
            // Test 1: Echo command
            System.out.println("=== Test 1: Echo Command ===");
            TestProtocol.EchoResponse echoResp = client.echo("Hello MediaManager!");
            System.out.println("Echo response: " + echoResp.getMessage());
            System.out.println("Server timestamp: " + echoResp.getServerTimestamp());
            System.out.println();
            
            // Test 2: Heartbeat command
            System.out.println("=== Test 2: Heartbeat Command ===");
            long startTime = System.currentTimeMillis();
            TestProtocol.HeartbeatResponse hbResp = client.heartbeat();
            long endTime = System.currentTimeMillis();
            
            long roundTripTime = endTime - startTime;
            System.out.println("Client timestamp: " + hbResp.getClientTimestamp());
            System.out.println("Server timestamp: " + hbResp.getServerTimestamp());
            System.out.println("Round-trip time: " + roundTripTime + "ms");
            System.out.println();
            
            // Test 3: Multiple echo commands
            System.out.println("=== Test 3: Multiple Echo Commands ===");
            for (int i = 1; i <= 5; i++) {
                TestProtocol.EchoResponse resp = client.echo("Message " + i);
                System.out.println(i + ". " + resp.getMessage());
            }
            System.out.println();
            
            // Test 4: Close command
            System.out.println("=== Test 4: Close Command ===");
            TestProtocol.CloseResponse closeResp = client.close();
            System.out.println("Close response: " + closeResp.getMessage());
            System.out.println();
            
        } catch (Exception e) {
            System.err.println("Error: " + e.getMessage());
            e.printStackTrace();
        } finally {
            // Disconnect
            try {
                client.disconnect();
            } catch (Exception e) {
                System.err.println("Error disconnecting: " + e.getMessage());
            }
        }
    }
}
```

---

## Step 5: Build and Run

### Build the Project
```bash
# Compile everything
mvn clean package

# Check that JAR was created
ls target/
# Should show: mediamanager-client-1.0-SNAPSHOT.jar
```

### Run the Client

First, make sure MediaManager Core is running:
```bash
# In another terminal, start MediaManager Core
cd /path/to/MediaManager-core
java -jar target/mediamanager-core.jar
```

Then run your client:
```bash
# Run with default socket path
java -cp target/mediamanager-client-1.0-SNAPSHOT.jar com.example.client.Main

# Or specify custom socket path
java -cp target/mediamanager-client-1.0-SNAPSHOT.jar com.example.client.Main /custom/path/mediamanager.sock
```

### Expected Output
```
Connecting to MediaManager Core...
Connected to MediaManager Core at: /tmp/mediamanager.sock

=== Test 1: Echo Command ===
Sending request a1b2c3d4-... (25 bytes)
Receiving response (37 bytes)
Received response a1b2c3d4-... (status: 200)
Echo response: Hello MediaManager!
Server timestamp: 1700000000000

=== Test 2: Heartbeat Command ===
Sending request e5f6g7h8-... (18 bytes)
Receiving response (28 bytes)
Received response e5f6g7h8-... (status: 200)
Client timestamp: 1700000000000
Server timestamp: 1700000000001
Round-trip time: 2ms

=== Test 3: Multiple Echo Commands ===
Sending request ...
1. Message 1
2. Message 2
3. Message 3
4. Message 4
5. Message 5

=== Test 4: Close Command ===
Sending request ...
Close response: Goodbye!

Disconnected from MediaManager Core
```

---

## Advanced Features

### Error Handling

Add robust error handling:
```java
public class RobustMediaManagerClient extends MediaManagerClient {
    
    private int maxRetries = 3;
    private int retryDelayMs = 1000;
    
    public RobustMediaManagerClient(String socketPath) {
        super(socketPath);
    }
    
    /**
     * Echo with automatic retry on failure
     */
    public TestProtocol.EchoResponse echoWithRetry(String message) throws IOException {
        int attempt = 0;
        Exception lastException = null;
        
        while (attempt < maxRetries) {
            try {
                return echo(message);
            } catch (IOException e) {
                lastException = e;
                attempt++;
                
                if (attempt < maxRetries) {
                    System.err.println("Attempt " + attempt + " failed, retrying...");
                    try {
                        Thread.sleep(retryDelayMs);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw new IOException("Interrupted during retry", ie);
                    }
                    
                    // Try to reconnect
                    try {
                        disconnect();
                        connect();
                    } catch (IOException reconnectEx) {
                        // Ignore, will retry in next iteration
                    }
                }
            }
        }
        
        throw new IOException("Failed after " + maxRetries + " attempts", lastException);
    }
}
```

### Connection Pooling

For high-throughput applications:
```java
public class ConnectionPool {
    
    private final BlockingQueue<MediaManagerClient> pool;
    private final String socketPath;
    private final int maxConnections;
    
    public ConnectionPool(String socketPath, int maxConnections) {
        this.socketPath = socketPath;
        this.maxConnections = maxConnections;
        this.pool = new LinkedBlockingQueue<>(maxConnections);
    }
    
    public MediaManagerClient acquire() throws IOException {
        MediaManagerClient client = pool.poll();
        
        if (client == null || !client.isConnected()) {
            client = new MediaManagerClient(socketPath);
            client.connect();
        }
        
        return client;
    }
    
    public void release(MediaManagerClient client) {
        if (client.isConnected() && pool.size() < maxConnections) {
            pool.offer(client);
        } else {
            try {
                client.disconnect();
            } catch (IOException e) {
                // Ignore
            }
        }
    }
    
    public void shutdown() {
        MediaManagerClient client;
        while ((client = pool.poll()) != null) {
            try {
                client.disconnect();
            } catch (IOException e) {
                // Ignore
            }
        }
    }
}
```

### Asynchronous Operations

For non-blocking operations:
```java
public class AsyncMediaManagerClient {
    
    private final MediaManagerClient client;
    private final ExecutorService executor;
    
    public AsyncMediaManagerClient(String socketPath) {
        this.client = new MediaManagerClient(socketPath);
        this.executor = Executors.newCachedThreadPool();
    }
    
    public CompletableFuture<TestProtocol.EchoResponse> echoAsync(String message) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                return client.echo(message);
            } catch (IOException e) {
                throw new CompletionException(e);
            }
        }, executor);
    }
    
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}

// Usage
AsyncMediaManagerClient asyncClient = new AsyncMediaManagerClient("/tmp/mediamanager.sock");

CompletableFuture<TestProtocol.EchoResponse> future = asyncClient.echoAsync("Hello!");

future.thenAccept(response -> {
    System.out.println("Echo: " + response.getMessage());
}).exceptionally(ex -> {
    System.err.println("Error: " + ex.getMessage());
    return null;
});
```

---

## Testing Your Client

### Unit Tests

Create `src/test/java/com/example/client/MediaManagerClientTest.java`:
```java
package com.example.client;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

public class MediaManagerClientTest {
    
    private MediaManagerClient client;
    
    @BeforeEach
    public void setUp() throws Exception {
        client = new MediaManagerClient("/tmp/mediamanager.sock");
        client.connect();
    }
    
    @AfterEach
    public void tearDown() throws Exception {
        if (client.isConnected()) {
            client.disconnect();
        }
    }
    
    @Test
    public void testEcho() throws Exception {
        var response = client.echo("Test message");
        assertEquals("Test message", response.getMessage());
        assertTrue(response.getServerTimestamp() > 0);
    }
    
    @Test
    public void testHeartbeat() throws Exception {
        var response = client.heartbeat();
        assertTrue(response.getClientTimestamp() > 0);
        assertTrue(response.getServerTimestamp() > 0);
    }
    
    @Test
    public void testMultipleRequests() throws Exception {
        for (int i = 0; i < 10; i++) {
            var response = client.echo("Message " + i);
            assertEquals("Message " + i, response.getMessage());
        }
    }
}
```

Add JUnit dependency to `pom.xml`:
```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>
```

Run tests:
```bash
mvn test
```

---

## Common Issues and Solutions

### Issue: "Connection refused"

**Problem:** MediaManager Core is not running

**Solution:**
```bash
# Check if Core is running
ps aux | grep mediamanager

# Start Core if not running
java -jar mediamanager-core.jar
```

### Issue: "Permission denied"

**Problem:** Socket file has restrictive permissions

**Solution:**
```bash
# Check permissions
ls -la /tmp/mediamanager.sock

# Should show: srw------- (owner only)
# If you're not the owner, run Core as your user
```

### Issue: "Invalid message size"

**Problem:** Protocol mismatch between client and server

**Solution:**
```bash
# Make sure proto files are identical
diff client/src/main/proto/transport.proto core/src/main/proto/transport.proto

# Regenerate sources if different
mvn clean generate-sources
```

### Issue: Timeout or hanging

**Problem:** Network buffer issues or deadlock

**Solution:**
```java
// Set socket timeout
channel.socket().setSoTimeout(5000); // 5 seconds

// Or use non-blocking mode
channel.configureBlocking(false);
```

---

## Best Practices

### Always Close Resources

Use try-with-resources or finally blocks:
```java
MediaManagerClient client = new MediaManagerClient("/tmp/mediamanager.sock");
try {
    client.connect();
    // Use client
} finally {
    client.disconnect();
}
```

### Validate Responses

Always check status codes:
```java
if (response.getStatusCode() != 200) {
    String error = response.getHeadersOrDefault("Error-Message", "Unknown error");
    throw new IOException("Server error: " + error);
}
```

### Log Protocol Details

During development, log all messages:
```java
System.out.println("Request: " + request.getRequestId());
System.out.println("Command: " + commandType);
System.out.println("Status: " + response.getStatusCode());
```

### Handle Network Errors Gracefully

Implement retry logic and reconnection:
```java
try {
    return client.echo(message);
} catch (IOException e) {
    // Log error
    logger.error("Communication error", e);
    
    // Attempt reconnect
    client.disconnect();
    client.connect();
    
    // Retry once
    return client.echo(message);
}
```

---

## Next Steps

Congratulations! You've built a working MediaManager client. Next steps:

1. **Add more commands** - Implement search, scan, organize operations
2. **Build a GUI** - Create a JavaFX or web interface
3. **Study server internals** - [[Server Architecture]]
4. **Explore gRPC** - [[gRPC Communication]] for remote access
5. **Debug issues** - [[Testing and Debugging]]

---

## Complete Example Repository

A complete working example is available in the MediaManager Core repository:
```bash
git clone https://git.gustavomiranda.xyz/GHMiranda/MediaManager-core
cd MediaManager-core/examples/java-client
mvn clean package
java -jar target/example-client.jar
```

---

**Ready to build more advanced features?** Check out [[Command Reference]] for all available operations!
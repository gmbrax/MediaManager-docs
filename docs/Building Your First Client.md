# Building Your First Client

This is a quick-start tutorial to build your first MediaManager client in under 10 minutes. We'll create a simple Java client that connects to MediaManager Core and sends an Echo command.

---

## Prerequisites

Before starting, make sure you have:

1. **Java 17+** installed - See [[Development Environment Setup]]
2. **Maven** installed
3. **MediaManager Core running** on your system
4. **Basic Java knowledge**

---

## What We'll Build

A simple command-line application that:
1. Connects to MediaManager Core via Unix socket
2. Sends an Echo command with a message
3. Receives and displays the response
4. Disconnects gracefully

**Total code:** Less than 100 lines of Java

---

## Step 1: Create Project Structure

Create a new directory and basic Maven project:
```bash
# Create project directory
mkdir my-first-client
cd my-first-client

# Create source directories
mkdir -p src/main/java/com/example
mkdir -p src/main/proto
```

---

## Step 2: Copy Protocol Buffer Files

Copy the `.proto` files from MediaManager Core:
```bash
# Copy from MediaManager Core repository
cp /path/to/MediaManager-core/src/main/proto/transport.proto src/main/proto/
cp /path/to/MediaManager-core/src/main/proto/test.proto src/main/proto/
```

Your structure should look like:
```
my-first-client/
├── src/
│   └── main/
│       ├── java/
│       │   └── com/example/
│       └── proto/
│           ├── transport.proto
│           └── test.proto
└── pom.xml (we'll create this next)
```

---

## Step 3: Create pom.xml

Create `pom.xml` in the project root:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-first-client</artifactId>
    <version>1.0</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>3.24.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:3.24.0:exe:${os.detected.classifier}</protocArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
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

---

## Step 4: Generate Protocol Buffer Classes
```bash
# Generate Java classes from .proto files
mvn clean generate-sources

# Verify generated files
ls target/generated-sources/protobuf/java/com/mediamanager/protocol/
```

You should see:
- `TransportProtocol.java`
- `TestProtocol.java`

---

## Step 5: Write the Client Code

Create `src/main/java/com/example/SimpleClient.java`:
```java
package com.example;

import com.google.protobuf.ByteString;
import com.mediamanager.protocol.TransportProtocol;
import com.mediamanager.protocol.TestProtocol;

import java.net.StandardProtocolFamily;
import java.net.UnixDomainSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.nio.file.Path;
import java.util.UUID;

public class SimpleClient {
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== MediaManager Simple Client ===\n");
        
        // Step 1: Connect to MediaManager Core
        System.out.println("Connecting to MediaManager Core...");
        Path socketPath = Path.of("/tmp/mediamanager.sock");
        UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketPath);
        SocketChannel channel = SocketChannel.open(StandardProtocolFamily.UNIX);
        channel.connect(address);
        System.out.println("Connected!\n");
        
        // Step 2: Create Echo command
        System.out.println("Creating Echo command...");
        TestProtocol.EchoCommand echoCmd = TestProtocol.EchoCommand.newBuilder()
            .setMessage("Hello MediaManager!")
            .build();
        
        // Step 3: Wrap in Request
        TransportProtocol.Request request = TransportProtocol.Request.newBuilder()
            .setRequestId(UUID.randomUUID().toString())
            .setPayload(ByteString.copyFrom(echoCmd.toByteArray()))
            .putHeaders("Command-Type", "Echo")
            .build();
        System.out.println("Request ID: " + request.getRequestId() + "\n");
        
        // Step 4: Send request with length-prefix
        System.out.println("Sending request...");
        byte[] requestBytes = request.toByteArray();
        ByteBuffer sendBuffer = ByteBuffer.allocate(4 + requestBytes.length);
        sendBuffer.putInt(requestBytes.length);  // Size header
        sendBuffer.put(requestBytes);            // Payload
        sendBuffer.flip();
        
        while (sendBuffer.hasRemaining()) {
            channel.write(sendBuffer);
        }
        System.out.println("Request sent (" + requestBytes.length + " bytes)\n");
        
        // Step 5: Read response size
        System.out.println("Reading response...");
        ByteBuffer sizeBuffer = ByteBuffer.allocate(4);
        while (sizeBuffer.hasRemaining()) {
            channel.read(sizeBuffer);
        }
        sizeBuffer.flip();
        int responseSize = sizeBuffer.getInt();
        System.out.println("Response size: " + responseSize + " bytes");
        
        // Step 6: Read response payload
        ByteBuffer responseBuffer = ByteBuffer.allocate(responseSize);
        while (responseBuffer.hasRemaining()) {
            channel.read(responseBuffer);
        }
        responseBuffer.flip();
        
        byte[] responseBytes = new byte[responseSize];
        responseBuffer.get(responseBytes);
        
        // Step 7: Parse response
        TransportProtocol.Response response = TransportProtocol.Response.parseFrom(responseBytes);
        System.out.println("Response ID: " + response.getRequestId());
        System.out.println("Status Code: " + response.getStatusCode() + "\n");
        
        // Step 8: Parse Echo response
        if (response.getStatusCode() == 200) {
            TestProtocol.EchoResponse echoResp = TestProtocol.EchoResponse.parseFrom(
                response.getPayload()
            );
            
            System.out.println("=== SUCCESS ===");
            System.out.println("Echo message: " + echoResp.getMessage());
            System.out.println("Server timestamp: " + echoResp.getServerTimestamp());
        } else {
            System.err.println("Error: " + response.getStatusCode());
            System.err.println(response.getHeadersOrDefault("Error-Message", "Unknown error"));
        }
        
        // Step 9: Disconnect
        System.out.println("\nDisconnecting...");
        channel.close();
        System.out.println("Done!");
    }
}
```

---

## Step 6: Compile and Run

### Compile
```bash
mvn clean package
```

You should see:
```
[INFO] BUILD SUCCESS
```

### Make Sure Core is Running

In another terminal, start MediaManager Core:
```bash
cd /path/to/MediaManager-core
java -jar target/mediamanager-core.jar
```

### Run Your Client
```bash
java -cp target/my-first-client-1.0.jar com.example.SimpleClient
```

---

## Expected Output
```
=== MediaManager Simple Client ===

Connecting to MediaManager Core...
Connected!

Creating Echo command...
Request ID: 550e8400-e29b-41d4-a716-446655440000

Sending request...
Request sent (42 bytes)

Reading response...
Response size: 54 bytes
Response ID: 550e8400-e29b-41d4-a716-446655440000
Status Code: 200

=== SUCCESS ===
Echo message: Hello MediaManager!
Server timestamp: 1700000000000

Disconnecting...
Done!
```

---

## What Just Happened?

Let's break down what your client did:

### 1. Connected to Unix Socket
```java
SocketChannel channel = SocketChannel.open(StandardProtocolFamily.UNIX);
channel.connect(UnixDomainSocketAddress.of(socketPath));
```
Opened a Unix Domain Socket connection to MediaManager Core.

### 2. Created a Command
```java
TestProtocol.EchoCommand echoCmd = TestProtocol.EchoCommand.newBuilder()
    .setMessage("Hello MediaManager!")
    .build();
```
Built an Echo command using Protocol Buffers.

### 3. Wrapped in Request
```java
TransportProtocol.Request request = TransportProtocol.Request.newBuilder()
    .setRequestId(UUID.randomUUID().toString())
    .setPayload(ByteString.copyFrom(echoCmd.toByteArray()))
    .putHeaders("Command-Type", "Echo")
    .build();
```
Wrapped the command in a generic Request message with metadata.

### 4. Sent with Length-Prefix
```java
ByteBuffer sendBuffer = ByteBuffer.allocate(4 + requestBytes.length);
sendBuffer.putInt(requestBytes.length);  // 4-byte size
sendBuffer.put(requestBytes);            // Payload
```
Sent the message with a 4-byte size header (length-prefix framing).

### 5. Received Response
```java
// Read 4-byte size
ByteBuffer sizeBuffer = ByteBuffer.allocate(4);
channel.read(sizeBuffer);
int responseSize = sizeBuffer.getInt();

// Read payload
ByteBuffer responseBuffer = ByteBuffer.allocate(responseSize);
channel.read(responseBuffer);
```
Read the response using the same length-prefix protocol.

### 6. Parsed Result
```java
TransportProtocol.Response response = TransportProtocol.Response.parseFrom(responseBytes);
TestProtocol.EchoResponse echoResp = TestProtocol.EchoResponse.parseFrom(response.getPayload());
```
Deserialized the Protocol Buffer messages.

---

## Try These Modifications

### 1. Send Different Messages

Change the echo message:
```java
TestProtocol.EchoCommand echoCmd = TestProtocol.EchoCommand.newBuilder()
    .setMessage("Your custom message here!")
    .build();
```

### 2. Send Multiple Requests

Add a loop to send multiple echo commands:
```java
for (int i = 1; i <= 5; i++) {
    TestProtocol.EchoCommand echoCmd = TestProtocol.EchoCommand.newBuilder()
        .setMessage("Message " + i)
        .build();
    
    // ... send and receive ...
    
    System.out.println(i + ". " + echoResp.getMessage());
}
```

### 3. Try Heartbeat Command

Replace Echo with Heartbeat:
```java
// Create Heartbeat command
TestProtocol.HeartbeatCommand hbCmd = TestProtocol.HeartbeatCommand.newBuilder()
    .setClientTimestamp(System.currentTimeMillis())
    .build();

// Wrap in Request
TransportProtocol.Request request = TransportProtocol.Request.newBuilder()
    .setRequestId(UUID.randomUUID().toString())
    .setPayload(ByteString.copyFrom(hbCmd.toByteArray()))
    .putHeaders("Command-Type", "Heartbeat")
    .build();

// ... send and receive ...

// Parse Heartbeat response
TestProtocol.HeartbeatResponse hbResp = TestProtocol.HeartbeatResponse.parseFrom(
    response.getPayload()
);

System.out.println("Client time: " + hbResp.getClientTimestamp());
System.out.println("Server time: " + hbResp.getServerTimestamp());
```

---

## Common Issues

### "Connection refused"

**Problem:** MediaManager Core is not running

**Solution:**
```bash
# Check if Core is running
ps aux | grep mediamanager

# Start Core
cd /path/to/MediaManager-core
java -jar target/mediamanager-core.jar
```

### "No such file or directory: /tmp/mediamanager.sock"

**Problem:** Socket file doesn't exist

**Solution:** Make sure MediaManager Core is running and creating the socket.

### "Permission denied"

**Problem:** You don't have permission to access the socket

**Solution:**
```bash
# Check socket permissions
ls -la /tmp/mediamanager.sock

# Should show: srw------- (owner read/write only)
# Run Core as your user or adjust permissions
```

### Build fails with "protoc not found"

**Problem:** Protocol Buffers compiler not installed

**Solution:** See [[Development Environment Setup]] for installation instructions.

---

## Understanding the Code Structure

Your simple client has three main parts:

### 1. Connection Setup
```java
SocketChannel channel = SocketChannel.open(StandardProtocolFamily.UNIX);
channel.connect(address);
```
Opens a Unix socket connection.

### 2. Request/Response Cycle
```java
// Build → Serialize → Send
Request → bytes → socket

// Receive → Deserialize → Parse
socket → bytes → Response
```

### 3. Length-Prefix Framing
```
[4 bytes: size][N bytes: protobuf message]
```
Simple protocol to delimit messages.

---

## Next Steps

Congratulations! You've built your first MediaManager client. Now you can:

### Learn More
- **[[Protocol Buffers Guide]]** - Deep dive into protobuf messages
- **[[Unix Socket Communication]]** - Understand the wire protocol
- **[[Client Development Guide]]** - Build a production-ready client

### Build More Features
- Add error handling and retries
- Implement connection pooling
- Create a command-line interface
- Build a GUI with JavaFX

### Explore Other Commands
- **Heartbeat** - Measure latency
- **Close** - Graceful disconnection
- **Search** - Query music library (when implemented)
- **Scan** - Trigger library scan (when implemented)

---

## Complete Code

The complete working code is available here:
```bash
# Clone example repository
git clone https://git.gustavomiranda.xyz/GHMiranda/MediaManager-core
cd MediaManager-core/examples/simple-client

# Build and run
mvn clean package
java -jar target/simple-client.jar
```

---

## Summary

You learned how to:
- Set up a Maven project with Protocol Buffers
- Connect to MediaManager Core via Unix socket
- Create and send commands using protobuf
- Implement length-prefix framing
- Receive and parse responses
- Handle basic errors

**Total time:** Less than 10 minutes

**Lines of code:** Less than 100

**Result:** A working MediaManager client!

---

**Ready for more advanced features?** Continue to [[Client Development Guide]] for a comprehensive tutorial!
# Unix Socket Communication

This guide explains how MediaManager Core uses **Unix Domain Sockets** for high-performance local Inter-Process Communication (IPC). You'll learn the wire protocol, message framing, and how to implement clients that communicate with the Core.

---

## What are Unix Domain Sockets?

Unix Domain Sockets are a **data communications endpoint** for exchanging data between processes executing on the same host operating system. Unlike TCP/IP sockets that communicate over a network, Unix sockets use the **filesystem** as their address space.

### Key Characteristics

**File-Based Addressing:**
- Socket appears as a file in the filesystem (e.g., `/tmp/mediamanager/mediamanager.sock`)
- Access controlled by standard file permissions
- No network stack overhead

**Performance:**
- **10-20x faster** than TCP on localhost
- Zero network protocol overhead
- Direct kernel memory-to-memory transfer
- No marshalling/unmarshalling of network headers

**Security:**
- File permission-based access control (e.g., `rw-------` = owner only)
- No network exposure - completely local
- Cannot be accessed remotely without SSH tunneling
- Process-level isolation

**Reliability:**
- Connection-oriented (like TCP)
- Guaranteed message ordering
- Automatic connection management

---

## MediaManager Socket Location

MediaManager Core creates its Unix socket at:
```
/tmp/mediamanager/mediamanager.sock
```

**Configuration:**
The socket path is configurable via `config.properties`:
```properties
ipc.socket.path=/tmp
```

The full path becomes: `{ipc.socket.path}/mediamanager.sock`

**Permissions:**
The socket is created with `rw-------` (0600) permissions, meaning only the owner can read/write.

---

## Wire Protocol Overview

MediaManager uses a simple **length-prefixed binary protocol**:
```
┌─────────────┬────────────────────────┐
│  4 bytes    │   N bytes              │
│  (int32)    │   (protobuf message)   │
│  Size       │   Payload              │
└─────────────┴────────────────────────┘
```

### Protocol Rules

1. **Every message** starts with a 4-byte (32-bit) integer indicating payload size
2. Size is encoded in **big-endian** (network byte order)
3. Size indicates **only the payload length**, not including the 4-byte header
4. After the size header comes the **serialized Protocol Buffer message**
5. Messages are **back-to-back** - no delimiters or padding

### Why Length-Prefixed?

**Problem:** TCP/Unix sockets are **stream-oriented**, not message-oriented. Data arrives as a continuous byte stream without natural boundaries.

**Solution:** Length-prefix tells the receiver exactly how many bytes to read for each message.

**Example without length-prefix:**
```
[Message1][Message2][Message3]
Where does Message1 end? Unknown!
```

**Example with length-prefix:**
```
[14][Message1bytes][10][Message2by][8][Message3]
     ← 14 bytes →      ← 10 bytes→   ← 8 bytes→
```

---

## Message Flow

### Client to Server (Request)

**Step 1:** Client creates a `Request` protobuf message
```java
Request request = Request.newBuilder()
    .setRequestId(UUID.randomUUID().toString())
    .setPayload(commandBytes)
    .putHeaders("Command-Type", "Echo")
    .build();
```

**Step 2:** Serialize to bytes
```java
byte[] messageBytes = request.toByteArray();
int messageSize = messageBytes.length;
```

**Step 3:** Create length-prefixed frame
```java
ByteBuffer buffer = ByteBuffer.allocate(4 + messageSize);
buffer.putInt(messageSize);  // 4-byte size header
buffer.put(messageBytes);    // Protobuf payload
buffer.flip();
```

**Step 4:** Write to socket
```java
while (buffer.hasRemaining()) {
    channel.write(buffer);
}
```

### Server to Client (Response)

**Step 1:** Server reads 4-byte size header
```java
ByteBuffer sizeBuffer = ByteBuffer.allocate(4);
while (sizeBuffer.hasRemaining()) {
    channel.read(sizeBuffer);
}
sizeBuffer.flip();
int messageSize = sizeBuffer.getInt();
```

**Step 2:** Server reads exact payload bytes
```java
ByteBuffer messageBuffer = ByteBuffer.allocate(messageSize);
while (messageBuffer.hasRemaining()) {
    channel.read(messageBuffer);
}
messageBuffer.flip();
```

**Step 3:** Deserialize protobuf
```java
byte[] messageBytes = new byte[messageSize];
messageBuffer.get(messageBytes);
Response response = Response.parseFrom(messageBytes);
```

**Step 4:** Process response
```java
if (response.getStatusCode() == 200) {
    // Success - process payload
} else {
    // Error - check headers for error message
}
```

---

## Java Implementation (Client Side)

Here's a complete example of a Java client connecting to MediaManager Core:

### Connecting to the Socket
```java
import java.net.StandardProtocolFamily;
import java.net.UnixDomainSocketAddress;
import java.nio.channels.SocketChannel;
import java.nio.file.Path;

public class MediaManagerClient {
    
    private SocketChannel channel;
    
    public void connect() throws IOException {
        // Define socket path
        Path socketPath = Path.of("/tmp/mediamanager/mediamanager.sock");
        UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketPath);
        
        // Open Unix Domain Socket channel
        channel = SocketChannel.open(StandardProtocolFamily.UNIX);
        
        // Connect to MediaManager Core
        channel.connect(address);
        
        System.out.println("Connected to MediaManager Core");
    }
    
    public void disconnect() throws IOException {
        if (channel != null && channel.isOpen()) {
            channel.close();
        }
    }
}
```

### Sending a Request
```java
import com.mediamanager.protocol.TransportProtocol;
import com.google.protobuf.ByteString;
import java.nio.ByteBuffer;
import java.util.UUID;

public void sendRequest(byte[] commandPayload, String commandType) throws IOException {
    // Build Request message
    TransportProtocol.Request request = TransportProtocol.Request.newBuilder()
        .setRequestId(UUID.randomUUID().toString())
        .setPayload(ByteString.copyFrom(commandPayload))
        .putHeaders("Command-Type", commandType)
        .build();
    
    // Serialize to bytes
    byte[] requestBytes = request.toByteArray();
    int messageSize = requestBytes.length;
    
    // Create buffer with size header + payload
    ByteBuffer buffer = ByteBuffer.allocate(4 + messageSize);
    buffer.putInt(messageSize);      // Length prefix
    buffer.put(requestBytes);        // Protobuf message
    buffer.flip();
    
    // Write to socket
    while (buffer.hasRemaining()) {
        channel.write(buffer);
    }
    
    System.out.println("Sent request: " + request.getRequestId());
}
```

### Receiving a Response
```java
public TransportProtocol.Response receiveResponse() throws IOException {
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
    
    // Validate size (security check)
    if (messageSize <= 0 || messageSize > 1024 * 1024) { // Max 1MB
        throw new IOException("Invalid message size: " + messageSize);
    }
    
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
    
    System.out.println("Received response: " + response.getRequestId());
    return response;
}
```

### Complete Echo Example
```java
import com.mediamanager.protocol.TestProtocol;

public class EchoExample {
    
    public static void main(String[] args) throws Exception {
        MediaManagerClient client = new MediaManagerClient();
        
        try {
            // Connect
            client.connect();
            
            // Create Echo command
            TestProtocol.EchoCommand echoCmd = TestProtocol.EchoCommand.newBuilder()
                .setMessage("Hello MediaManager!")
                .build();
            
            // Send request
            client.sendRequest(echoCmd.toByteArray(), "Echo");
            
            // Receive response
            TransportProtocol.Response response = client.receiveResponse();
            
            // Check status
            if (response.getStatusCode() == 200) {
                // Parse echo response
                TestProtocol.EchoResponse echoResp = 
                    TestProtocol.EchoResponse.parseFrom(response.getPayload());
                
                System.out.println("Echo: " + echoResp.getMessage());
                System.out.println("Server time: " + echoResp.getServerTimestamp());
            } else {
                System.err.println("Error: " + response.getStatusCode());
                System.err.println(response.getHeadersOrDefault("Error-Message", "Unknown"));
            }
            
        } finally {
            client.disconnect();
        }
    }
}
```

---

## Server Implementation (Core Side)

Here's how MediaManager Core handles the socket communication:

### Server Socket Setup
```java
import java.net.StandardProtocolFamily;
import java.net.UnixDomainSocketAddress;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.file.Path;
import java.nio.file.Files;
import java.nio.file.attribute.PosixFilePermissions;

public class IPCManager {
    
    private ServerSocketChannel serverChannel;
    private Path socketPath;
    
    public void init() throws Exception {
        // Define socket path
        socketPath = Path.of("/tmp/mediamanager/mediamanager.sock");
        
        // Delete existing socket if present
        if (Files.exists(socketPath)) {
            Files.deleteIfExists(socketPath);
        }
        
        // Create socket address
        UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketPath);
        
        // Open server socket channel
        serverChannel = ServerSocketChannel.open(StandardProtocolFamily.UNIX);
        serverChannel.bind(address);
        
        // Set permissions (owner read/write only)
        Files.setPosixFilePermissions(socketPath, 
            PosixFilePermissions.fromString("rw-------"));
        
        // Configure non-blocking mode
        serverChannel.configureBlocking(false);
        
        System.out.println("Server listening on: " + socketPath);
    }
}
```

### Accepting Connections
```java
private void acceptConnections() {
    while (running.get()) {
        try {
            // Accept returns null if no client (non-blocking)
            SocketChannel clientChannel = serverChannel.accept();
            
            if (clientChannel != null) {
                System.out.println("Client connected");
                
                // Handle client in thread pool
                clientThreadPool.submit(() -> handleClient(clientChannel));
            } else {
                // No client, sleep briefly
                Thread.sleep(1);
            }
            
        } catch (IOException e) {
            if (running.get()) {
                System.err.println("Error accepting connection: " + e.getMessage());
            }
        }
    }
}
```

### Reading Requests
```java
private TransportProtocol.Request readRequest(SocketChannel channel) throws IOException {
    // Read size header (4 bytes)
    ByteBuffer sizeBuffer = ByteBuffer.allocate(4);
    int bytesRead = 0;
    
    while (bytesRead < 4) {
        int read = channel.read(sizeBuffer);
        if (read == -1) {
            return null; // Client disconnected
        }
        bytesRead += read;
    }
    
    sizeBuffer.flip();
    int messageSize = sizeBuffer.getInt();
    
    // Validate size
    if (messageSize <= 0 || messageSize > 1024 * 1024) { // Max 1MB
        throw new IOException("Invalid message size: " + messageSize);
    }
    
    // Read payload
    ByteBuffer messageBuffer = ByteBuffer.allocate(messageSize);
    bytesRead = 0;
    
    while (bytesRead < messageSize) {
        int read = channel.read(messageBuffer);
        if (read == -1) {
            throw new IOException("Connection closed while reading");
        }
        bytesRead += read;
    }
    
    messageBuffer.flip();
    
    // Deserialize
    byte[] messageBytes = new byte[messageSize];
    messageBuffer.get(messageBytes);
    
    return TransportProtocol.Request.parseFrom(messageBytes);
}
```

### Writing Responses
```java
private void writeResponse(SocketChannel channel, TransportProtocol.Response response) 
        throws IOException {
    // Serialize
    byte[] responseBytes = response.toByteArray();
    int messageSize = responseBytes.length;
    
    // Create buffer with size + payload
    ByteBuffer buffer = ByteBuffer.allocate(4 + messageSize);
    buffer.putInt(messageSize);
    buffer.put(responseBytes);
    buffer.flip();
    
    // Write to socket
    while (buffer.hasRemaining()) {
        channel.write(buffer);
    }
}
```

### Client Handler Loop
```java
private void handleClient(SocketChannel channel) {
    try {
        while (channel.isOpen()) {
            // Read request
            TransportProtocol.Request request = readRequest(channel);
            
            if (request == null) {
                // Client disconnected
                break;
            }
            
            // Process request (via DelegateActionManager)
            TransportProtocol.Response response = processRequest(request);
            
            // Send response
            writeResponse(channel, response);
            
            // Check if connection should close
            String connectionHeader = response.getHeadersOrDefault("Connection", "");
            if ("close".equals(connectionHeader)) {
                break;
            }
        }
    } catch (IOException e) {
        System.err.println("Client error: " + e.getMessage());
    } finally {
        try {
            channel.close();
        } catch (IOException e) {
            // Ignore
        }
    }
}
```

---

## Python Implementation

For Python clients, use the built-in `socket` module:

### Connecting and Sending
```python
import socket
import struct
from transport_pb2 import Request, Response
from test_pb2 import EchoCommand, EchoResponse
import uuid

def connect_to_mediamanager():
    # Create Unix socket
    sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    
    # Connect to MediaManager
    sock.connect('/tmp/mediamanager/mediamanager.sock')
    
    return sock

def send_request(sock, command_payload, command_type):
    # Build Request
    request = Request()
    request.request_id = str(uuid.uuid4())
    request.payload = command_payload
    request.headers['Command-Type'] = command_type
    
    # Serialize
    request_bytes = request.SerializeToString()
    message_size = len(request_bytes)
    
    # Send size header (4 bytes, big-endian)
    size_header = struct.pack('>I', message_size)
    sock.sendall(size_header)
    
    # Send payload
    sock.sendall(request_bytes)
    
    print(f"Sent request: {request.request_id}")
    return request.request_id

def receive_response(sock):
    # Read size header
    size_data = sock.recv(4)
    if len(size_data) < 4:
        raise Exception("Connection closed")
    
    message_size = struct.unpack('>I', size_data)[0]
    
    # Validate size
    if message_size <= 0 or message_size > 1024 * 1024:
        raise Exception(f"Invalid message size: {message_size}")
    
    # Read payload
    payload_data = b''
    while len(payload_data) < message_size:
        chunk = sock.recv(message_size - len(payload_data))
        if not chunk:
            raise Exception("Connection closed while reading")
        payload_data += chunk
    
    # Deserialize
    response = Response()
    response.ParseFromString(payload_data)
    
    print(f"Received response: {response.request_id}")
    return response

# Example usage
def main():
    sock = connect_to_mediamanager()
    
    try:
        # Create echo command
        echo_cmd = EchoCommand()
        echo_cmd.message = "Hello from Python!"
        
        # Send request
        send_request(sock, echo_cmd.SerializeToString(), "Echo")
        
        # Receive response
        response = receive_response(sock)
        
        if response.status_code == 200:
            echo_resp = EchoResponse()
            echo_resp.ParseFromString(response.payload)
            print(f"Echo: {echo_resp.message}")
            print(f"Server time: {echo_resp.server_timestamp}")
        else:
            print(f"Error: {response.status_code}")
            
    finally:
        sock.close()

if __name__ == '__main__':
    main()
```

---

## Go Implementation

For Go clients:
```go
package main

import (
    "encoding/binary"
    "fmt"
    "io"
    "net"
    
    "google.golang.org/protobuf/proto"
    pb "path/to/generated/proto"
)

func connectToMediaManager() (net.Conn, error) {
    conn, err := net.Dial("unix", "/tmp/mediamanager/mediamanager.sock")
    if err != nil {
        return nil, err
    }
    return conn, nil
}

func sendRequest(conn net.Conn, commandPayload []byte, commandType string) error {
    // Build Request
    request := &pb.Request{
        RequestId: generateUUID(),
        Payload:   commandPayload,
        Headers: map[string]string{
            "Command-Type": commandType,
        },
    }
    
    // Serialize
    requestBytes, err := proto.Marshal(request)
    if err != nil {
        return err
    }
    
    messageSize := uint32(len(requestBytes))
    
    // Send size header (4 bytes, big-endian)
    sizeHeader := make([]byte, 4)
    binary.BigEndian.PutUint32(sizeHeader, messageSize)
    
    if _, err := conn.Write(sizeHeader); err != nil {
        return err
    }
    
    // Send payload
    if _, err := conn.Write(requestBytes); err != nil {
        return err
    }
    
    fmt.Printf("Sent request: %s\n", request.RequestId)
    return nil
}

func receiveResponse(conn net.Conn) (*pb.Response, error) {
    // Read size header
    sizeHeader := make([]byte, 4)
    if _, err := io.ReadFull(conn, sizeHeader); err != nil {
        return nil, err
    }
    
    messageSize := binary.BigEndian.Uint32(sizeHeader)
    
    // Validate size
    if messageSize == 0 || messageSize > 1024*1024 {
        return nil, fmt.Errorf("invalid message size: %d", messageSize)
    }
    
    // Read payload
    payloadData := make([]byte, messageSize)
    if _, err := io.ReadFull(conn, payloadData); err != nil {
        return nil, err
    }
    
    // Deserialize
    response := &pb.Response{}
    if err := proto.Unmarshal(payloadData, response); err != nil {
        return nil, err
    }
    
    fmt.Printf("Received response: %s\n", response.RequestId)
    return response, nil
}

func main() {
    conn, err := connectToMediaManager()
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    
    // Create echo command
    echoCmd := &pb.EchoCommand{
        Message: "Hello from Go!",
    }
    
    cmdBytes, _ := proto.Marshal(echoCmd)
    
    // Send request
    if err := sendRequest(conn, cmdBytes, "Echo"); err != nil {
        panic(err)
    }
    
    // Receive response
    response, err := receiveResponse(conn)
    if err != nil {
        panic(err)
    }
    
    if response.StatusCode == 200 {
        echoResp := &pb.EchoResponse{}
        proto.Unmarshal(response.Payload, echoResp)
        fmt.Printf("Echo: %s\n", echoResp.Message)
        fmt.Printf("Server time: %d\n", echoResp.ServerTimestamp)
    }
}
```

---

## Error Handling

### Common Error Scenarios

**Connection Refused:**
```
Error: Connection refused
```
**Cause:** MediaManager Core is not running
**Solution:** Start the Core daemon

**Permission Denied:**
```
Error: Permission denied
```
**Cause:** Socket file has restrictive permissions
**Solution:** Check socket permissions with `ls -la /tmp/mediamanager/mediamanager.sock`

**Invalid Message Size:**
```
Error: Invalid message size: 0 or > 1MB
```
**Cause:** Corrupted data or protocol mismatch
**Solution:** Verify protobuf schemas match between client and server

**Connection Reset:**
```
Error: Connection reset by peer
```
**Cause:** Server crashed or closed connection unexpectedly
**Solution:** Check server logs, implement reconnection logic

### Best Practices

**Always validate message size:**
```java
if (messageSize <= 0 || messageSize > MAX_MESSAGE_SIZE) {
    throw new IOException("Invalid message size");
}
```

**Handle partial reads:**
```java
while (bytesRead < targetSize) {
    int read = channel.read(buffer);
    if (read == -1) {
        throw new IOException("Unexpected EOF");
    }
    bytesRead += read;
}
```

**Implement timeouts:**
```java
channel.socket().setSoTimeout(5000); // 5 second timeout
```

**Check status codes:**
```java
if (response.getStatusCode() != 200) {
    String errorMsg = response.getHeadersOrDefault("Error-Message", "Unknown error");
    throw new RuntimeException("Server error: " + errorMsg);
}
```

---

## Performance Considerations

### Buffer Sizing

**Small messages** (< 1KB): Allocate exact size
```java
ByteBuffer buffer = ByteBuffer.allocate(4 + messageSize);
```

**Large messages** (> 100KB): Use buffer pools or streaming
```java
// Reuse buffers
ByteBuffer buffer = bufferPool.acquire();
try {
    // Use buffer
} finally {
    bufferPool.release(buffer);
}
```

### Connection Pooling

For high-throughput scenarios, maintain a pool of connections:
```java
public class ConnectionPool {
    private BlockingQueue<SocketChannel> pool;
    private int maxConnections = 10;
    
    public SocketChannel acquire() throws Exception {
        SocketChannel channel = pool.poll();
        if (channel == null || !channel.isConnected()) {
            channel = createConnection();
        }
        return channel;
    }
    
    public void release(SocketChannel channel) {
        if (channel.isConnected() && pool.size() < maxConnections) {
            pool.offer(channel);
        } else {
            try {
                channel.close();
            } catch (IOException e) {
                // Ignore
            }
        }
    }
}
```

### Reducing Allocations

**Reuse ByteBuffers:**
```java
// Instead of allocating each time
ByteBuffer buffer = ByteBuffer.allocate(4096);

// Reuse
buffer.clear();
channel.read(buffer);
buffer.flip();
```

**Use Direct Buffers** for better performance:
```java
ByteBuffer buffer = ByteBuffer.allocateDirect(4096);
```

---

## Testing and Debugging

### Testing with `socat`

You can test the socket manually using `socat`:
```bash
# Connect to socket
socat - UNIX-CONNECT:/tmp/mediamanager/mediamanager.sock

# Or create a test server
socat UNIX-LISTEN:/tmp/test.sock -
```

### Inspecting Socket Traffic

Use `strace` to see socket operations:
```bash
# Trace a client process
strace -e trace=connect,read,write,sendto,recvfrom ./your-client

# Output shows:
# connect(3, {sa_family=AF_UNIX, sun_path="/tmp/mediamanager/mediamanager.sock"}, 110) = 0
# write(3, "\0\0\0\x1a...", 30) = 30
# read(3, "\0\0\0\x15...", 4) = 4
```

### Logging Binary Data

Log message frames in hex for debugging:
```java
private void logFrame(String direction, byte[] data) {
    StringBuilder hex = new StringBuilder();
    for (byte b : data) {
        hex.append(String.format("%02x ", b));
    }
    System.out.println(direction + ": " + hex.toString());
}

// Usage
logFrame("SEND", buffer.array());
logFrame("RECV", messageBytes);
```

---

## Security Considerations

### File Permissions

**Always verify socket permissions:**
```bash
ls -la /tmp/mediamanager/mediamanager.sock
# Should show: srw------- (owner read/write only)
```

**Set permissions in code:**
```java
Files.setPosixFilePermissions(socketPath, 
    PosixFilePermissions.fromString("rw-------"));
```

### Input Validation

**Always validate message sizes:**
```java
private static final int MAX_MESSAGE_SIZE = 1024 * 1024; // 1MB

if (messageSize > MAX_MESSAGE_SIZE) {
    throw new SecurityException("Message too large");
}
```

**Validate request IDs:**
```java
if (request.getRequestId() == null || request.getRequestId().isEmpty()) {
    return createErrorResponse("Missing request ID", 400);
}
```

### Denial of Service Prevention

**Implement rate limiting:**
```java
private RateLimiter rateLimiter = RateLimiter.create(100.0); // 100 req/sec

if (!rateLimiter.tryAcquire()) {
    return createErrorResponse("Rate limit exceeded", 429);
}
```

**Set connection limits:**
```java
private Semaphore connectionSemaphore = new Semaphore(50); // Max 50 connections

if (!connectionSemaphore.tryAcquire()) {
    channel.close();
    return;
}
```

---

## Summary

### Key Takeaways

**Protocol Structure:**
- 4-byte length prefix (big-endian int32)
- Followed by Protocol Buffer message
- No delimiters or padding between messages

**Implementation Steps:**
1. Connect to `/tmp/mediamanager/mediamanager.sock`
2. Build Request protobuf with command payload
3. Serialize Request to bytes
4. Send 4-byte size + payload
5. Read 4-byte size from response
6. Read exact payload bytes
7. Deserialize Response protobuf
8. Process based on status code

**Performance:**
- Use buffer pooling for high throughput
- Reuse ByteBuffers to reduce allocations
- Consider connection pooling for many requests
- Direct buffers for better I/O performance

**Security:**
- Validate all message sizes
- Check file permissions (rw-------)
- Implement rate limiting
- Handle errors gracefully

---

## Next Steps

Now that you understand Unix Socket communication:

1. **Build a complete client** → [[Client Development Guide]]
2. **Explore server internals** → [[Server Architecture]]
3. **Learn available commands** → [[Command Reference]]
4. **Debug issues** → [[Testing and Debugging]]

---

**Ready to build your client?** Continue to [[Client Development Guide]] for a complete walkthrough!
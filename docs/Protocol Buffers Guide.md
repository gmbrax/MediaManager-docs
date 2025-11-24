# Protocol Buffers Guide

## What are Protocol Buffers?

**Protocol Buffers** (protobuf) is a language-neutral, platform-neutral, extensible mechanism for serializing structured data. Developed by Google, it's like JSON or XML, but smaller, faster, and generates native language bindings.

In MediaManager, Protocol Buffers serve as the **contract** between the Core and any client application, enabling efficient binary communication regardless of the programming language used.

---

## Why Protocol Buffers?

### Performance

| Format | Size (bytes) | Serialization (µs) | Deserialization (µs) |
|--------|--------------|-------------------|---------------------|
| **Protobuf** | 28 | 0.1 | 0.1 |
| JSON | 82 | 1.5 | 2.3 |
| XML | 124 | 3.2 | 4.1 |

*Example: Serializing a simple message with 5 fields*

### Key Advantages

**Compact Binary Format**
- 3-10x smaller than JSON
- Reduced network bandwidth
- Faster I/O operations

**Strongly Typed**
- Compile-time type checking
- No runtime parsing errors
- IDE autocomplete support

**Language Agnostic**
- Same `.proto` file generates code for Java, Python, Go, C++, Rust, JavaScript, etc.
- Consistent API across languages
- Perfect for polyglot architectures

**Backward/Forward Compatible**
- Add fields without breaking old clients
- Remove fields safely
- Version your APIs gracefully

**Code Generation**
- No manual serialization code
- No boilerplate
- Focus on business logic

---

## MediaManager Protocol Buffers Architecture

MediaManager uses a **layered message structure**:

**Layer 1: Application Layer**
- Specific commands (SearchCommand, EchoCommand, etc)
- Business logic messages

**Layer 2: Transport Layer**
- Request/Response wrapper messages
- Generic envelope for all commands

**Layer 3: Wire Protocol**
- 4 bytes size + protobuf data
- Length-prefixed framing

This design provides:
- **Flexibility**: Add new commands without changing transport layer
- **Extensibility**: Multiple command types share same infrastructure
- **Simplicity**: Consistent request/response pattern

---

## MediaManager .proto Files

MediaManager Core defines its protocol in `.proto` files located in `src/main/proto/`:

### File Structure
```
src/main/proto/
├── transport.proto      # Core Request/Response messages
└── test.proto          # Example commands (Echo, Heartbeat, Close)
```

Let's explore each file in detail.

---

## transport.proto - Core Protocol

This file defines the **fundamental communication structure** used by all MediaManager clients.

### Full File Content
```protobuf
syntax = "proto3";

option java_package = "com.mediamanager.protocol";
option java_outer_classname = "TransportProtocol";

package mediamanager;

message Request {
  string request_id = 1;
  bytes payload = 2;
  map<string, string> headers = 3;
}

message Response {
  string request_id = 1;
  int32 status_code = 2;
  bytes payload = 3;
  map<string, string> headers = 4;
}
```

### Line-by-Line Breakdown

#### Syntax Declaration
```protobuf
syntax = "proto3";
```
- Specifies Protocol Buffers version 3
- Proto3 is simpler and more modern than proto2
- Required as the first non-comment line

#### Java Options
```protobuf
option java_package = "com.mediamanager.protocol";
option java_outer_classname = "TransportProtocol";
```
- `java_package`: Generated Java classes will be in this package
- `java_outer_classname`: Container class name for all messages
- Result: `com.mediamanager.protocol.TransportProtocol.Request`

#### Package Declaration
```protobuf
package mediamanager;
```
- Namespace to prevent naming conflicts
- Used by other languages (Python: `mediamanager_pb2`, Go: `mediamanager`)

---

### Request Message
```protobuf
message Request {
  string request_id = 1;
  bytes payload = 2;
  map<string, string> headers = 3;
}
```

#### Field Breakdown

**1. request_id (field number 1)**
```protobuf
string request_id = 1;
```
- **Type**: `string`
- **Purpose**: Unique identifier for this request (typically UUID)
- **Usage**: Match requests with responses in async scenarios
- **Example**: `"550e8400-e29b-41d4-a716-446655440000"`

**2. payload (field number 2)**
```protobuf
bytes payload = 2;
```
- **Type**: `bytes` (arbitrary binary data)
- **Purpose**: Contains the **serialized command** (e.g., SearchCommand, EchoCommand)
- **Why bytes?**: Maximum flexibility - any protobuf message can be serialized here
- **Pattern**: Nested message serialization

**3. headers (field number 3)**
```protobuf
map<string, string> headers = 3;
```
- **Type**: `map<string, string>` (key-value pairs)
- **Purpose**: Metadata about the request
- **Common headers**:
  - `Command-Type`: "Echo", "Search", "Scan", etc.
  - `Client-Version`: "1.0.0"
  - `Auth-Token`: (for authenticated requests)
- **Example**:
```json
  {
    "Command-Type": "Echo",
    "Client-Version": "1.0.0"
  }
```

#### Field Numbers

The numbers (`1`, `2`, `3`) are **critical**:
- They identify fields in the binary format
- **Never change** field numbers once deployed
- Can skip numbers: `1, 2, 5, 10` is valid
- Range 1-15: Use 1 byte to encode (most efficient)
- Range 16-2047: Use 2 bytes
- **Reserved ranges**: 19000-19999, 536870912-536870911

---

### Response Message
```protobuf
message Response {
  string request_id = 1;
  int32 status_code = 2;
  bytes payload = 3;
  map<string, string> headers = 4;
}
```

#### Field Breakdown

**1. request_id (field number 1)**
```protobuf
string request_id = 1;
```
- **Matches** the `request_id` from the Request
- Enables request/response correlation
- Essential for async processing

**2. status_code (field number 2)**
```protobuf
int32 status_code = 2;
```
- **Type**: 32-bit signed integer
- **Purpose**: HTTP-like status codes
- **Common values**:
  - `200`: Success
  - `400`: Bad Request (invalid command)
  - `404`: Not Found (e.g., track doesn't exist)
  - `500`: Internal Server Error
  - `201`: Created (e.g., new playlist)

**3. payload (field number 3)**
```protobuf
bytes payload = 3;
```
- **Contains**: Serialized response data (e.g., SearchResults, EchoResponse)
- **Type**: `bytes` for flexibility
- **Empty on error**: Check `status_code` first

**4. headers (field number 4)**
```protobuf
map<string, string> headers = 4;
```
- **Response metadata**:
  - `Connection`: "close" (close connection after this response)
  - `Server-Version`: "1.0.0"
  - `Processing-Time-Ms`: "42"
  - `Error-Message`: (if status_code indicates error)

---

## test.proto - Example Commands

This file defines **example commands** for testing and learning:

### Full File Content
```protobuf
syntax = "proto3";

option java_package = "com.mediamanager.protocol";
option java_outer_classname = "TestProtocol";

package mediamanager.test;

message EchoCommand {
  string message = 1;
}

message EchoResponse {
  string message = 1;
  int64 server_timestamp = 2;
}

message HeartbeatCommand {
  int64 client_timestamp = 1;
}

message HeartbeatResponse {
  int64 client_timestamp = 1;
  int64 server_timestamp = 2;
}

message CloseCommand {
  // Empty - just signals connection close
}

message CloseResponse {
  string message = 1;
}
```

---

### EchoCommand & EchoResponse

**Purpose**: Simple request-response test
```protobuf
message EchoCommand {
  string message = 1;
}
```
- Send a string, get it echoed back
- Useful for testing connectivity
```protobuf
message EchoResponse {
  string message = 1;        // Echoed message
  int64 server_timestamp = 2; // Server time in milliseconds
}
```
- Includes server timestamp for latency calculation

**Usage Example**:
```java
// Client sends
EchoCommand cmd = EchoCommand.newBuilder()
    .setMessage("Hello MediaManager!")
    .build();

// Server responds
EchoResponse resp = EchoResponse.newBuilder()
    .setMessage("Hello MediaManager!")
    .setServerTimestamp(System.currentTimeMillis())
    .build();
```

---

### HeartbeatCommand & HeartbeatResponse

**Purpose**: Keep-alive and latency measurement
```protobuf
message HeartbeatCommand {
  int64 client_timestamp = 1;  // Client time when sent
}

message HeartbeatResponse {
  int64 client_timestamp = 1;  // Echoed client time
  int64 server_timestamp = 2;  // Server time when received
}
```

**Latency Calculation**:
```java
long sendTime = System.currentTimeMillis();
// ... send heartbeat ...
// ... receive response ...
long receiveTime = System.currentTimeMillis();

long roundTripTime = receiveTime - sendTime;
long oneWayLatency = roundTripTime / 2;

System.out.println("RTT: " + roundTripTime + "ms");
System.out.println("Latency: " + oneWayLatency + "ms");
```

---

### CloseCommand & CloseResponse

**Purpose**: Graceful connection termination
```protobuf
message CloseCommand {
  // Empty - just signals close
}

message CloseResponse {
  string message = 1;  // Goodbye message
}
```

**Flow**:
1. Client sends CloseCommand
2. Server responds with CloseResponse
3. Server sets `Connection: close` header
4. Both sides close the socket

---

## How Commands Work with Transport

Commands are **nested** inside Request/Response messages:

### Sending a Command
```java
// 1. Create your command
EchoCommand command = EchoCommand.newBuilder()
    .setMessage("Test")
    .build();

// 2. Serialize it to bytes
byte[] commandBytes = command.toByteArray();

// 3. Wrap in Request
Request request = Request.newBuilder()
    .setRequestId(UUID.randomUUID().toString())
    .setPayload(ByteString.copyFrom(commandBytes))  // Command as bytes
    .putHeaders("Command-Type", "Echo")
    .build();

// 4. Send request (covered in Unix Socket Communication page)
```

### Receiving a Response
```java
// 1. Receive Response (covered in Unix Socket Communication page)
Response response = /* ... received from socket ... */;

// 2. Check status code
if (response.getStatusCode() != 200) {
    System.err.println("Error: " + response.getHeadersOrDefault("Error-Message", "Unknown"));
    return;
}

// 3. Extract payload
byte[] payloadBytes = response.getPayload().toByteArray();

// 4. Deserialize to your expected message type
EchoResponse echoResponse = EchoResponse.parseFrom(payloadBytes);

// 5. Use the data
System.out.println("Echo: " + echoResponse.getMessage());
System.out.println("Server time: " + echoResponse.getServerTimestamp());
```

---

## Compiling Protocol Buffers

Protocol Buffer files (`.proto`) must be **compiled** into language-specific code.

### Java Compilation (Maven)

MediaManager Core uses Maven with the protobuf plugin.

**pom.xml excerpt**:
```xml
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
    </plugins>
</build>
```

**Compile**:
```bash
# Generate Java classes from .proto files
mvn generate-sources

# Generated files location
ls target/generated-sources/protobuf/java/com/mediamanager/protocol/
# TransportProtocol.java
# TestProtocol.java
```

---

### Python Compilation
```bash
# Install protobuf compiler support for Python
pip install protobuf grpcio-tools

# Compile .proto files
python -m grpc_tools.protoc \
    -I./src/main/proto \
    --python_out=./python-client \
    ./src/main/proto/transport.proto \
    ./src/main/proto/test.proto

# Generated files
ls python-client/
# transport_pb2.py
# test_pb2.py
```

**Usage**:
```python
import transport_pb2
import test_pb2

# Create request
request = transport_pb2.Request()
request.request_id = "12345"
request.headers["Command-Type"] = "Echo"

echo_cmd = test_pb2.EchoCommand()
echo_cmd.message = "Hello from Python"
request.payload = echo_cmd.SerializeToString()

# Serialize for transmission
data = request.SerializeToString()
```

---

### JavaScript/TypeScript Compilation
```bash
# Install dependencies
npm install google-protobuf
npm install -D @types/google-protobuf

# Compile with protoc (install protoc-gen-js first)
protoc \
    --js_out=import_style=commonjs,binary:./js-client \
    --proto_path=./src/main/proto \
    ./src/main/proto/transport.proto \
    ./src/main/proto/test.proto

# Generated files
ls js-client/
# transport_pb.js
# test_pb.js
```

**Usage**:
```javascript
const { Request } = require('./transport_pb');
const { EchoCommand } = require('./test_pb');

// Create command
const echo = new EchoCommand();
echo.setMessage("Hello from JS");

// Create request
const request = new Request();
request.setRequestId("12345");
request.setPayload(echo.serializeBinary());
request.getHeadersMap().set("Command-Type", "Echo");

// Serialize
const bytes = request.serializeBinary();
```

---

### Go Compilation
```bash
# Install protoc-gen-go
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# Compile
protoc \
    --go_out=./go-client \
    --go_opt=paths=source_relative \
    --proto_path=./src/main/proto \
    ./src/main/proto/transport.proto \
    ./src/main/proto/test.proto

# Generated files
ls go-client/
# transport.pb.go
# test.pb.go
```

**Usage**:
```go
package main

import (
    "github.com/google/uuid"
    pb "path/to/generated/proto"
)

func main() {
    // Create command
    echo := &pb.EchoCommand{
        Message: "Hello from Go",
    }
    
    // Serialize command
    cmdBytes, _ := proto.Marshal(echo)
    
    // Create request
    request := &pb.Request{
        RequestId: uuid.New().String(),
        Payload:   cmdBytes,
        Headers: map[string]string{
            "Command-Type": "Echo",
        },
    }
    
    // Serialize request
    data, _ := proto.Marshal(request)
}
```

---

### Rust Compilation
```bash
# Add to Cargo.toml
# [dependencies]
# prost = "0.12"
# prost-types = "0.12"
#
# [build-dependencies]
# prost-build = "0.12"

# Create build.rs
cat > build.rs << 'EOF'
fn main() {
    prost_build::compile_protos(
        &["src/main/proto/transport.proto", "src/main/proto/test.proto"],
        &["src/main/proto/"]
    ).unwrap();
}
EOF

# Build (generates code automatically)
cargo build
```

**Usage**:
```rust
use prost::Message;

// Generated structs available
let echo = EchoCommand {
    message: "Hello from Rust".to_string(),
};

// Serialize
let mut cmd_bytes = Vec::new();
echo.encode(&mut cmd_bytes).unwrap();

// Create request
let request = Request {
    request_id: uuid::Uuid::new_v4().to_string(),
    payload: cmd_bytes,
    headers: [("Command-Type".to_string(), "Echo".to_string())]
        .iter().cloned().collect(),
};

// Serialize request
let mut data = Vec::new();
request.encode(&mut data).unwrap();
```

---

## Protocol Buffers Data Types

Understanding protobuf types is essential:

### Scalar Types

| Proto Type | Java | Python | Go | JavaScript | Rust |
|------------|------|--------|-----|-----------|------|
| `double` | double | float | float64 | number | f64 |
| `float` | float | float | float32 | number | f32 |
| `int32` | int | int | int32 | number | i32 |
| `int64` | long | int | int64 | number/string | i64 |
| `uint32` | int | int | uint32 | number | u32 |
| `uint64` | long | int | uint64 | number/string | u64 |
| `bool` | boolean | bool | bool | boolean | bool |
| `string` | String | str | string | string | String |
| `bytes` | ByteString | bytes | []byte | Uint8Array | Vec<u8> |

### Complex Types

**Message** (nested structure):
```protobuf
message Artist {
    string name = 1;
    string country = 2;
}

message Track {
    string title = 1;
    Artist artist = 2;  // Nested message
}
```

**Repeated** (arrays/lists):
```protobuf
message SearchResults {
    repeated Track tracks = 1;  // List of tracks
}
```

**Map** (key-value):
```protobuf
message Metadata {
    map<string, string> tags = 1;  // e.g., {"genre": "Rock", "year": "1969"}
}
```

**Enum**:
```protobuf
enum Format {
    UNKNOWN = 0;  // First value must be 0
    MP3 = 1;
    FLAC = 2;
    AAC = 3;
}

message Track {
    string title = 1;
    Format format = 2;
}
```

**Oneof** (union type):
```protobuf
message Command {
    oneof command_type {
        EchoCommand echo = 1;
        SearchCommand search = 2;
        ScanCommand scan = 3;
    }
}
```

---

## Versioning and Evolution

One of protobuf's greatest strengths is **schema evolution**.

### Rules for Backward Compatibility

**Safe Changes**:
- **Add new fields** (old clients ignore them)
- **Delete fields** (mark as `reserved`)
- **Rename fields** (field numbers stay the same)
- **Change field type** (sometimes, e.g., int32 to int64)

**Breaking Changes**:
- **Change field numbers** (corrupts data)
- **Change field type incompatibly** (e.g., string to int32)
- **Change `repeated` to non-repeated**

### Example: Adding a Field

**Version 1**:
```protobuf
message SearchCommand {
    string query = 1;
}
```

**Version 2** (add limit):
```protobuf
message SearchCommand {
    string query = 1;
    int32 limit = 2;  // New field
}
```

**Result**:
- Old clients: Send `query` only, server uses default `limit=0`
- New clients: Send both `query` and `limit`
- **No breaking changes**

### Removing Fields Safely

**Version 1**:
```protobuf
message Track {
    string title = 1;
    string deprecated_field = 2;
    string artist = 3;
}
```

**Version 2** (remove deprecated_field):
```protobuf
message Track {
    string title = 1;
    reserved 2;  // Reserve field number
    reserved "deprecated_field";  // Reserve field name
    string artist = 3;
}
```

**Why reserve?**
- Prevents accidental reuse of field number 2
- Avoids data corruption if old messages still exist

---

## Best Practices

### 1. Use Field Numbers Wisely
```protobuf
message Track {
    // Most frequently used fields: 1-15 (1 byte encoding)
    string title = 1;
    string artist = 2;
    string album = 3;
    
    // Less common fields: 16+ (2 bytes)
    int64 duration_ms = 16;
    int32 bitrate = 17;
    string file_path = 18;
}
```

### 2. Always Use Explicit Field Names
```protobuf
// Bad - unclear
message Response {
    int32 code = 1;
    bytes data = 2;
}

// Good - self-documenting
message Response {
    int32 status_code = 1;
    bytes payload = 2;
}
```

### 3. Document Complex Messages
```protobuf
// Represents a music track with metadata
message Track {
    // Unique identifier (UUID)
    string id = 1;
    
    // Display title (e.g., "Abbey Road")
    string title = 2;
    
    // Primary artist name
    string artist = 3;
    
    // Duration in milliseconds
    int64 duration_ms = 4;
}
```

### 4. Use Enums for Fixed Sets
```protobuf
// Good
enum AudioFormat {
    AUDIO_FORMAT_UNKNOWN = 0;
    AUDIO_FORMAT_MP3 = 1;
    AUDIO_FORMAT_FLAC = 2;
}

message Track {
    AudioFormat format = 1;
}

// Bad
message Track {
    string format = 1;  // "mp3", "flac", etc. - error-prone
}
```

### 5. Reserve Deleted Fields
```protobuf
message Track {
    reserved 2, 5 to 7;  // Reserved field numbers
    reserved "old_field", "deprecated";  // Reserved names
    
    string title = 1;
    string artist = 3;
}
```

---

## Debugging Protocol Buffers

### View Binary Data (protoc)
```bash
# Serialize a message to file
echo '{"request_id":"123","payload":"test"}' | \
    protoc --encode=mediamanager.Request transport.proto > request.bin

# View as text
protoc --decode=mediamanager.Request transport.proto < request.bin
```

### Inspect with xxd
```bash
# View binary protobuf data in hex
xxd request.bin

# Example output:
# 00000000: 0a03 3132 3312 0474 6573 74              ..123..test
```

### Python Inspection
```python
import transport_pb2

# Parse binary data
with open('request.bin', 'rb') as f:
    request = transport_pb2.Request()
    request.ParseFromString(f.read())
    
print(request)  # Human-readable output
```

---

## Real-World Example: Search Command

Let's create a realistic search command:

### Define the Message

**commands.proto** (new file):
```protobuf
syntax = "proto3";

option java_package = "com.mediamanager.protocol";
option java_outer_classname = "CommandProtocol";

package mediamanager.commands;

message SearchCommand {
    string query = 1;
    int32 limit = 2;
    int32 offset = 3;
    repeated string fields = 4;  // Search in: "title", "artist", "album"
}

message Track {
    string id = 1;
    string title = 2;
    string artist = 3;
    string album = 4;
    int64 duration_ms = 5;
}

message SearchResults {
    repeated Track tracks = 1;
    int32 total_count = 2;
    int32 offset = 3;
}
```

### Java Usage
```java
// Client: Create search command
SearchCommand search = SearchCommand.newBuilder()
    .setQuery("Beatles")
    .setLimit(10)
    .setOffset(0)
    .addFields("title")
    .addFields("artist")
    .build();

// Wrap in Request
Request request = Request.newBuilder()
    .setRequestId(UUID.randomUUID().toString())
    .setPayload(search.toByteString())
    .putHeaders("Command-Type", "Search")
    .build();

// Send via socket...

// Server: Parse and respond
Request receivedRequest = Request.parseFrom(socketData);
SearchCommand cmd = SearchCommand.parseFrom(receivedRequest.getPayload());

// Execute search...
List<Track> results = database.search(cmd.getQuery(), cmd.getLimit());

// Build response
SearchResults searchResults = SearchResults.newBuilder()
    .addAllTracks(results)
    .setTotalCount(results.size())
    .setOffset(cmd.getOffset())
    .build();

Response response = Response.newBuilder()
    .setRequestId(receivedRequest.getRequestId())
    .setStatusCode(200)
    .setPayload(searchResults.toByteString())
    .build();

// Send back to client...
```

---

## Summary

### Key Takeaways

**Protocol Buffers are the contract** between MediaManager Core and clients

**Language-agnostic** - Write `.proto` once, use in any language

**Two-layer architecture**:
- `transport.proto`: Generic Request/Response wrapper
- `test.proto` / `commands.proto`: Specific commands

**Efficient and typed** - Better than JSON for IPC

**Evolution-friendly** - Add fields without breaking compatibility

**Code generation** - No manual serialization needed

### Files in MediaManager

| File | Purpose | Generated Class |
|------|---------|-----------------|
| `transport.proto` | Core protocol (Request/Response) | `TransportProtocol.java` |
| `test.proto` | Example commands | `TestProtocol.java` |

---

## Next Steps

Now that you understand Protocol Buffers:

1. **Learn the wire protocol** - [[Unix Socket Communication]] - How messages are framed and sent
2. **Build a client** - [[Client Development Guide]] - Put it all together
3. **Explore examples** - [[Code Examples]] - Working implementations

---

**Ready to see how these messages flow over Unix Sockets?** Continue to [[Unix Socket Communication]]!
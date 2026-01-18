---
description: Architectural reference for Status
---

# Status

**Package:** com.hypixel.hytale.protocol.packets.auth
**Type:** Transient Data Model

## Definition
```java
// Signature
public class Status implements Packet {
```

## Architecture & Concepts
The Status class is a data model representing a network packet, specifically the server status response. It conforms to the Packet interface, identifying itself with a static PACKET ID of 10. Its primary role is to act as a Data Transfer Object (DTO) for conveying basic server information—such as its name, message of the day (MOTD), and player counts—to a client before a full gameplay connection is established. This is a fundamental component of the server discovery and listing feature.

Architecturally, this class encapsulates the complex binary serialization format required by the Hytale protocol. It is not a service or a manager; it is a pure data container whose logic is exclusively concerned with translating its state to and from a Netty ByteBuf.

The binary layout is a high-performance design that separates data into a fixed-size block and a variable-size block.
- **Fixed Block:** Contains a bitmask for nullable fields (`nullBits`) and fixed-width integers (`playerCount`, `maxPlayers`).
- **Variable Block:** Contains the actual data for variable-length fields like `name` and `motd`. The fixed block contains offsets pointing to the location of this data within the variable block. This structure minimizes payload size by omitting null fields entirely from the variable block.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side (Sending):** Instantiated directly via `new Status(...)` when the server needs to reply to a status request. The server's current state is passed into the constructor.
    - **Client-Side (Receiving):** Instantiated by the network protocol layer, which calls the static `deserialize` factory method. A client-side developer will almost never create an instance of this class directly.
- **Scope:** Extremely short-lived and request-scoped. An instance exists only for the brief moment it is being serialized into a network buffer or being read by a network event handler.
- **Destruction:** The object becomes eligible for garbage collection as soon as the serialization or handling logic completes. It holds no managed resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Mutable. All data fields are public and can be modified after construction. This design facilitates easy population of the object before it is passed to the network layer for serialization.
- **Thread Safety:** **This class is not thread-safe.** It contains no locks or other concurrency primitives.
    - **WARNING:** Sharing a Status instance across multiple threads is unsafe and will lead to data corruption and non-deterministic behavior. All interaction with a Status object must be confined to a single thread, typically a Netty I/O worker thread.

## API Surface
The primary contract of this class revolves around its static serialization and validation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static Status | O(N) | Constructs a Status object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the Hytale binary protocol. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check on a buffer to verify if it contains a structurally valid Status packet. Does not create an object. |
| computeSize() | int | O(N) | Calculates the total number of bytes this object will occupy when serialized. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total size of a packet directly from a buffer without full deserialization. |

*Complexity O(N) refers to the total length of the string fields.*

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol framework, not directly by application logic.

**Server-Side (Preparing a Response)**
```java
// The server populates a new Status packet with its current state.
Status serverStatus = new Status(
    "Hytale Adventure World",
    "Welcome! Now running on version X.Y.Z",
    42,
    200
);

// The packet is then passed to the Netty channel pipeline.
// An encoder in the pipeline will automatically call the serialize method.
channel.writeAndFlush(serverStatus);
```

**Client-Side (Handling a Received Packet)**
```java
// Inside a network event handler, after the pipeline has decoded the packet.
// The framework delivers a fully formed Status object.
public void onServerStatusReceived(Status status) {
    // Application logic consumes the data to update the UI.
    ServerListUI.updateEntry(serverId, status.name, status.motd, status.playerCount);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not cache and modify a Status object for multiple responses. They are lightweight DTOs and a new instance should be created for each discrete network operation to ensure data integrity.
- **Manual Serialization:** Avoid calling `serialize` or `deserialize` directly in application code. This is the responsibility of the network pipeline's encoders and decoders. Manual invocation risks severe buffer mismanagement, such as incorrect reader/writer indices or memory leaks.
- **Cross-Thread Sharing:** Never pass a Status instance from a network thread to another thread for modification. If data needs to be shared, copy the primitive values into a thread-safe application-level data structure.

## Data Pipeline
The Status packet is a simple data record that flows through the network stack.

**Outbound Flow (Server)**
> Flow:
> Server State (Player Count, MOTD) -> **new Status(...)** -> Netty Channel Pipeline (Encoder calls `serialize`) -> Raw ByteBuf -> TCP Socket

**Inbound Flow (Client)**
> Flow:
> TCP Socket -> Raw ByteBuf -> Netty Channel Pipeline (Decoder calls `deserialize`) -> **Status Instance** -> Network Event Handler -> Application Logic (e.g., UI Update)


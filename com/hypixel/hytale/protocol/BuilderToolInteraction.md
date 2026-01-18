---
description: Architectural reference for BuilderToolInteraction
---

# BuilderToolInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The BuilderToolInteraction class is a Data Transfer Object (DTO) that defines the wire format for a specific gameplay action within the Hytale network protocol. It is not a service or manager; it is a structured message designed for efficient serialization and deserialization over a network connection.

Its primary architectural feature is a custom, high-performance binary layout that minimizes payload size. This layout is divided into two main sections:

1.  **Fixed-Size Block:** A 19-byte header containing primitive data types like floats, booleans, and integers. This block has a predictable structure for fast, direct memory access.
2.  **Variable-Size Block:** A subsequent data region containing complex, nullable objects such as maps, arrays, and nested protocol objects. The fixed-size block contains integer offsets pointing to the start of each data structure within this variable region.

To further optimize for size, the protocol uses a single leading byte as a **bitfield** (named nullBits). Each bit in this field corresponds to a nullable, variable-sized field, indicating whether it is present in the payload. If a bit is zero, the corresponding data and its offset are omitted entirely, saving significant space for sparsely populated messages.

This design prioritizes network efficiency and deserialization speed over human readability. All operations are performed directly on Netty ByteBuf objects, avoiding intermediate data structures and minimizing memory allocations during network processing.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderToolInteraction is created in one of two scenarios:
    1.  **Inbound:** The static factory method `deserialize` is invoked by a network pipeline handler (e.g., a Netty decoder) when a corresponding packet ID is read from the network stream. This creates and populates the object from raw bytes.
    2.  **Outbound:** Game logic on the sending system (client or server) instantiates the class directly via its constructor to build a message that will be sent over the network.

-   **Scope:** The object's lifetime is extremely short and bound to the scope of a single network event or game tick. It is designed to be created, read, and then immediately discarded.

-   **Destruction:** The object holds no external resources and requires no explicit cleanup. It becomes eligible for garbage collection as soon as the game logic or network handler that received it completes its execution.

## Internal State & Concurrency

-   **State:** The object is fully **Mutable**. Its fields are populated after construction, primarily by the `deserialize` method. It is intended to be a simple data container.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit external synchronization. It is designed for synchronous processing within a single network event loop thread or the main game thread.

    **Warning:** Concurrent reads and writes will lead to unpredictable behavior and data corruption. Do not retain references to this object for processing on other threads. If multi-threaded processing is required, create a deep copy using the `clone` method for each thread.

## API Surface

The primary contract is defined by its static utility methods, which operate on network buffers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BuilderToolInteraction | O(N) | Constructs and populates an object from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into a ByteBuf and returns the number of bytes written. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized object in a buffer without allocating the object itself. Used for skipping records. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a security and integrity check on the buffer to ensure it contains a structurally valid object before attempting deserialization. |
| clone() | BuilderToolInteraction | O(N) | Creates a deep copy of the object, including all nested data structures. |

## Integration Patterns

### Standard Usage

**Deserialization (Receiving Data)**
The object is typically created by a protocol decoder and passed to a handler for game logic processing.

```java
// In a Netty ChannelInboundHandler or similar
void handlePacket(ByteBuf packetBuffer) {
    // Assume packetBuffer contains the data for one interaction,
    // starting at the readable index (offset 0).
    BuilderToolInteraction interaction = BuilderToolInteraction.deserialize(packetBuffer, 0);
    
    // Pass the fully-formed DTO to the game engine
    gameLogic.processBuilderInteraction(interaction);
}
```

**Serialization (Sending Data)**
Game logic creates and populates an instance before passing it to the network layer for encoding.

```java
// In a game system that sends data
void sendInteraction() {
    BuilderToolInteraction interaction = new BuilderToolInteraction();
    
    // Populate fields based on game state
    interaction.runTime = 1.5f;
    interaction.cancelOnItemChange = true;
    interaction.settings = new HashMap<>(); // ... and so on
    
    // Pass the object to the network layer to be serialized
    networkManager.send(interaction);
}
```

### Anti-Patterns (Do NOT do this)

-   **Object Reuse:** Do not reuse an instance of BuilderToolInteraction for multiple distinct messages. The `serialize` method assumes a clean state. Always create a new instance for each new outbound message.
-   **Asynchronous State Mutation:** Do not hold a reference to a deserialized object and modify it on another thread. The network thread may have already moved on, and the underlying ByteBuf could be released or recycled, leading to memory access violations.
-   **Manual Serialization:** Avoid calling `serialize` directly unless you are implementing a custom network encoder. The core network stack is responsible for wrapping the serialized payload with the correct packet ID, length prefixing, and other protocol framing.

## Data Pipeline

The BuilderToolInteraction class is a critical component in the flow of data between the network stack and the game logic.

**Inbound Data Flow (Receiving a Packet)**
> Flow:
> Raw TCP Stream -> Netty Channel -> Packet Framer (Length-Prefixed) -> Packet Decoder (ID-based) -> **BuilderToolInteraction.deserialize** -> Game Logic Handler

**Outbound Data Flow (Sending a Packet)**
> Flow:
> Game Event -> Game Logic -> **new BuilderToolInteraction()** -> Packet Encoder -> Packet Framer -> Netty Channel -> Raw TCP Stream


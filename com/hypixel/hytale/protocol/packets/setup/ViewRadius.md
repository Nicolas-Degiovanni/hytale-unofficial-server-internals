---
description: Architectural reference for the ViewRadius network packet.
---

# ViewRadius

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ViewRadius implements Packet {
```

## Architecture & Concepts
The ViewRadius class is a low-level Data Transfer Object (DTO) that represents a single, specific message within the Hytale network protocol stack. It is not a service or manager, but rather a concrete implementation of the **Packet** interface, defining the structure and serialization logic for packet ID 32.

Its primary role is to encapsulate the client's requested or the server's enforced view radius, a critical configuration parameter exchanged during the initial "setup" phase of a client-server connection. This class acts as a strongly-typed schema for a fixed-size block of 4 bytes on the wire, translating raw network data into a meaningful game concept.

The design pattern employed here is common in high-performance networking: static methods like deserialize and validateStructure operate directly on network buffers (Netty ByteBuf), allowing the core network pipeline to decode messages without needing to instantiate an object first. The object is only created upon successful validation and decoding.

## Lifecycle & Ownership
-   **Creation:**
    -   **Inbound (Receiving):** Instantiated by the protocol's central packet decoder when a message with ID 32 is read from the network stream. The static factory method `deserialize` is the designated entry point for this path.
    -   **Outbound (Sending):** Instantiated directly via `new ViewRadius(value)` by higher-level application logic, such as a session manager or server configuration service, before being passed to the network channel for encoding.
-   **Scope:** Extremely short-lived and ephemeral. An instance of ViewRadius typically exists only within the scope of a single network event handler method. Once its integer `value` is extracted and passed to the relevant game system, the object is no longer referenced and becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or resource release requirements.

## Internal State & Concurrency
-   **State:** Mutable. The core state is a single public integer field, `value`. This design prioritizes performance and direct access over encapsulation, which is acceptable for a short-lived, internal DTO. It contains no caches or derived state.
-   **Thread Safety:** **This class is not thread-safe.** The public, mutable `value` field makes it inherently unsafe for concurrent access. Packet objects are designed to be confined to a single network thread (e.g., a Netty I/O worker thread). They must not be shared across threads without external synchronization or by passing an immutable copy.

## API Surface
The public API is divided between instance methods for serialization and static methods for deserialization and validation, forming the complete contract for the network pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (32). |
| serialize(ByteBuf) | void | O(1) | Encodes the internal `value` into the provided buffer using little-endian byte order. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload in bytes (4). |
| deserialize(ByteBuf, int) | static ViewRadius | O(1) | **Factory Method.** Decodes a new ViewRadius instance from the given buffer at the specified offset. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check on a buffer to ensure enough readable bytes exist for a valid packet. |

## Integration Patterns

### Standard Usage
A developer will typically only instantiate this class when sending a message. When receiving, the object is provided by the network event system.

**Sending a ViewRadius Packet:**
```java
// Example: A server sending the required view radius to a client.
int serverEnforcedRadius = 16;
ViewRadius packet = new ViewRadius(serverEnforcedRadius);

// The networkChannel is an abstraction for the client connection.
// The pipeline automatically calls packet.serialize().
networkChannel.send(packet);
```

**Receiving a ViewRadius Packet:**
```java
// Example: A network handler method that processes inbound packets.
// The 'packet' object is delivered by the framework after deserialization.
public void handleViewRadius(ViewRadius packet) {
    int radius = packet.value;
    // Update the game's rendering system with the new radius.
    gameRenderer.setViewDistance(radius);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not modify and re-send the same ViewRadius instance. This can lead to unpredictable behavior if the object is still referenced elsewhere. Always create a new packet for each outbound message.
-   **Cross-Thread Sharing:** Never pass a ViewRadius instance from the network thread to a game logic thread. Extract the primitive `value` and pass it instead to avoid race conditions.
-   **Manual Serialization:** Avoid calling `serialize` directly in application code. The network pipeline's encoders are responsible for this task. Direct invocation circumvents the pipeline and can lead to corrupted data streams.

## Data Pipeline
The ViewRadius object is a transient data structure that exists at a specific point in the network data flow.

**Inbound Flow (Client or Server Receiving):**
> TCP Socket -> Netty ByteBuf -> Protocol Frame Decoder -> **ViewRadius.deserialize()** -> ViewRadius Instance -> Network Event Handler -> Game State Update

**Outbound Flow (Client or Server Sending):**
> Game Logic -> `new ViewRadius()` -> Network Channel Write -> Protocol Encoder -> **ViewRadius.serialize()** -> Netty ByteBuf -> TCP Socket


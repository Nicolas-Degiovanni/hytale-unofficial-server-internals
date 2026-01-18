---
description: Architectural reference for DisplayDebug
---

# DisplayDebug

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Data Transfer Object / Transient

## Definition
```java
// Signature
public class DisplayDebug implements Packet {
```

## Architecture & Concepts
The DisplayDebug class is a Data Transfer Object (DTO) that represents a single, specific network message: a command to render a debug visualization on the client. It is a fundamental component of the Hytale Protocol Layer, acting as both the in-memory representation and the serialization/deserialization schema for packet ID 114.

This class is not a service or manager; it is a pure data container. Its primary architectural function is to decouple the game logic that *requests* a debug shape from the low-level network systems that *transmit* that request.

The binary layout defined by this class is highly optimized for network performance. It employs a hybrid structure:
1.  **Null-Bit Field:** A single leading byte acts as a bitmask to indicate which of the nullable, variable-sized fields (like matrix and frustumProjection) are present in the payload. This allows the deserializer to skip entire data sections without parsing them.
2.  **Fixed-Size Block:** A contiguous block of memory for fields that are always present and have a known size, such as shape, time, and fade.
3.  **Variable-Size Block:** A final section containing the data for variable-length arrays. The fixed-size block contains integer offsets pointing to the start of each variable-length field within this block, enabling random access and preventing a full sequential scan.

This design minimizes payload size and deserialization overhead, which is critical for high-frequency network traffic.

## Lifecycle & Ownership
-   **Creation:**
    -   **Outbound (Sending Peer):** Instantiated directly via its constructor (`new DisplayDebug(...)`) by a system that needs to send a debug command. For example, a server-side developer tool might create one to visualize an AI's line of sight.
    -   **Inbound (Receiving Peer):** Instantiated by the protocol engine's central packet dispatcher. After the packet ID 114 is read from the network stream, the dispatcher invokes the static `DisplayDebug.deserialize` factory method, which constructs and populates the object from the raw ByteBuf.
-   **Scope:** Transient and extremely short-lived. An instance of DisplayDebug exists only for the brief period between its deserialization from the network buffer and the completion of its processing by a handler system (e.g., the client's rendering engine).
-   **Destruction:** The object is immediately eligible for garbage collection after its data has been consumed by the relevant system. There are no native resources or explicit cleanup methods associated with this class.

## Internal State & Concurrency
-   **State:** Mutable. All data-holding fields are public and can be modified after construction. The class is a simple data aggregate and performs no internal caching or state management. Its state is a direct 1-to-1 mapping of the data on the wire.
-   **Thread Safety:** **This class is not thread-safe.** It contains no locks or other concurrency primitives. It is designed under the assumption that it will be created and written on a single thread, or read and processed on a single thread (typically a Netty I/O thread or the main game thread).

    **Warning:** Concurrent modification and serialization of a DisplayDebug instance will result in corrupted network data, race conditions, and undefined behavior. Access must be externally synchronized if the object is shared across threads.

## API Surface
The public contract is dominated by static methods for validation and deserialization, and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static DisplayDebug | O(N) | Constructs a new DisplayDebug instance from a raw ByteBuf. N is the total length of all variable-sized arrays. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. Throws ProtocolException if an array exceeds maximum length. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs lightweight checks on a buffer to verify offsets and lengths without full deserialization. Critical for network security and stability. |
| computeSize() | int | O(1) | Calculates the total byte size the object will occupy when serialized. Useful for pre-allocating buffers. |

## Integration Patterns

### Standard Usage
A developer typically interacts with this class in one of two ways: creating it to be sent, or receiving it in a message handler. Direct calls to serialize or deserialize are managed by the protocol engine.

**Sending a Packet (Server-Side Logic):**
```java
// Example: Create a command to display a red sphere for 5 seconds.
Vector3f redColor = new Vector3f(1.0f, 0.0f, 0.0f);
DisplayDebug debugPacket = new DisplayDebug(DebugShape.Sphere, null, redColor, 5.0f, true, null);

// The network system is responsible for calling serialize() and sending the data.
playerConnection.sendPacket(debugPacket);
```

**Receiving a Packet (Client-Side Handler):**
```java
// In a network event handler method...
public void handleDisplayDebug(DisplayDebug packet) {
    // The packet is already deserialized by the engine.
    // Consume its data to render the shape.
    DebugRenderSystem.getInstance().addShape(
        packet.shape,
        packet.matrix,
        packet.color,
        packet.time
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not modify and re-send the same DisplayDebug instance. These objects are lightweight and should be treated as immutable once created. Re-using them can lead to unexpected state issues.
-   **Manual Serialization:** Never attempt to manually construct the byte payload. The binary format, with its null-bit field and variable offsets, is complex and must be handled by the `serialize` method to ensure correctness.
-   **Ignoring Validation:** On the receiving end, the protocol engine must call `validateStructure` before attempting to deserialize. Bypassing this step exposes the client or server to buffer overflow vulnerabilities from maliciously crafted packets.

## Data Pipeline
The primary flow for this object is its journey from a server-side command into a client-side rendering event.

> Flow:
> Server Logic -> `new DisplayDebug()` -> Network Engine -> **DisplayDebug.serialize()** -> TCP Stream -> Client Network Engine -> **DisplayDebug.deserialize()** -> Client Event Bus -> Debug Rendering System


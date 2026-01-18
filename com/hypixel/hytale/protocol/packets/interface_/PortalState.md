---
description: Architectural reference for PortalState
---

# PortalState

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class PortalState {
```

## Architecture & Concepts
The PortalState class is a Plain Old Java Object (POJO) that functions as a Data Transfer Object (DTO) within the Hytale network protocol. Its sole purpose is to encapsulate and transport the state of an in-game portal between the server and client.

This class is not a service or a manager; it is a pure data container. The presence of static serialization, deserialization, and validation methods, along with explicit binary layout constants like FIXED_BLOCK_SIZE, firmly establishes its role as a low-level component for network packet construction and parsing. It represents a "snapshot in time" of a portal's status, such as the time remaining before activation and whether it is actively being breached.

PortalState is designed to be embedded within larger, more complex network packets that manage user interface state or world events. It provides a standardized, fixed-size binary representation for portal information, ensuring consistent communication between endpoints.

## Lifecycle & Ownership
-   **Creation:** An instance of PortalState is created under two primary conditions:
    1.  **Deserialization:** The static factory method PortalState.deserialize is invoked by a higher-level packet decoder when parsing an incoming network stream from a Netty ByteBuf.
    2.  **Serialization:** Game logic on the sending side (typically the server) instantiates it directly (e.g., `new PortalState(seconds, isBreaching)`) to populate a packet that is about to be sent.

-   **Scope:** The object's lifetime is extremely short and tied to the scope of a single network operation. It exists only for the duration of packet processing. It is not intended to be cached or held as long-term state.

-   **Destruction:** The object is eligible for garbage collection as soon as the parent network packet has been fully processed or transmitted. There are no manual cleanup or `close` operations required.

## Internal State & Concurrency
-   **State:** The internal state is **fully mutable**. The public fields `remainingSeconds` and `breaching` can be modified at any time after creation. The class is a simple data holder and performs no internal validation or caching.

-   **Thread Safety:** This class is **not thread-safe**. It contains no locks, volatile keywords, or other concurrency primitives. Its public mutable fields make it inherently unsafe for concurrent access.

    **WARNING:** Instances of PortalState must be confined to the thread that created them, which is typically a Netty I/O worker thread or the main game logic thread. Sharing an instance across threads without external synchronization or deep cloning will result in undefined behavior and severe race conditions.

## API Surface
The primary contract is defined by the static methods responsible for protocol-level interactions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static PortalState | O(1) | Constructs a new PortalState instance by reading 5 bytes from the given buffer at the specified offset. |
| serialize(buf) | void | O(1) | Writes the object's state as 5 bytes into the provided buffer. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Checks if the buffer contains at least 5 readable bytes from the offset. Does not validate content. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 5 bytes. |

## Integration Patterns

### Standard Usage
PortalState is designed to be used by packet handlers during serialization and deserialization. It is almost never interacted with directly by high-level game systems.

```java
// Example: Deserializing from a parent packet's buffer
public void read(ByteBuf packetBuffer) {
    // ... read other packet data ...
    int portalStateOffset = ...; // Calculate offset within the buffer

    ValidationResult result = PortalState.validateStructure(packetBuffer, portalStateOffset);
    if (result.isError()) {
        throw new ProtocolException(result.getMessage());
    }

    PortalState portalInfo = PortalState.deserialize(packetBuffer, portalStateOffset);
    
    // Pass the deserialized data to the UI or game logic
    gameUI.updatePortalDisplay(portalInfo.remainingSeconds, portalInfo.breaching);
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term State:** Do not retain instances of PortalState in services, managers, or game entities as a means of storing state. It is a transfer object, not a state model. Convert its data into your application's domain model upon receipt.
-   **Cross-Thread Sharing:** Do not pass a PortalState instance from a network thread to a game logic thread. Instead, extract the primitive values (`int`, `boolean`) and pass those, or create a deep copy using the `clone` method or copy constructor.
-   **Manual Serialization:** Do not manually write integers and booleans to a buffer to mimic this object. Always use the provided `serialize` method to guarantee correctness and forward compatibility with the protocol.

## Data Pipeline
PortalState acts as the data payload within a larger network communication flow. It does not process data itself; it *is* the data being processed.

> **Flow (Server to Client):**
> Server Game Logic -> **new PortalState()** -> Parent Packet Serialization -> Netty Channel -> Client Network Layer -> Parent Packet Deserialization -> **PortalState.deserialize()** -> Client UI Update


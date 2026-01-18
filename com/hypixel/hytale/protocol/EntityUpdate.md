---
description: Architectural reference for EntityUpdate
---

# EntityUpdate

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class EntityUpdate {
```

## Architecture & Concepts

The EntityUpdate class is a fundamental Data Transfer Object (DTO) within Hytale's networking protocol, specifically designed to communicate state changes for a single entity between the server and client. It embodies a set of modifications—component additions, updates, and removals—that must be applied to a given entity, identified by its networkId.

This class is not a service or a manager; it is a pure data container. Its primary role is to act as a structured message format for the Entity Component System (ECS) synchronization layer.

The binary serialization format is highly optimized for network efficiency and parsing performance, built upon several key principles:

*   **Fixed and Variable Blocks:** The serialized payload is split into two sections. A small, fixed-size header (13 bytes) contains the entity's networkId and offsets to the variable-sized data. The variable data, containing the actual component updates, follows this header. This design allows a parser to quickly read the header and know exactly where to jump in the buffer to find specific data arrays, enabling efficient random access within the message.
*   **Nullable Bit Field:** The very first byte of the payload is a bitmask. Each bit corresponds to a nullable field (like `removed` or `updates`), indicating whether that field is present in the payload. This is a critical optimization that avoids wasting bandwidth for entities that only have components added but none removed, or vice-versa.
*   **VarInt Encoding:** To further conserve space, the counts for the `removed` and `updates` arrays are encoded using the VarInt format, which uses a variable number of bytes to represent an integer. Small numbers require only a single byte, which is the common case.

This class and its serialization logic are foundational to how the game world remains synchronized across the network.

## Lifecycle & Ownership

-   **Creation:** An EntityUpdate object is created under two primary circumstances:
    1.  **Server-Side (Serialization):** Instantiated by the server's game logic when an entity's state changes. The server populates the `updates` and `removed` fields before passing the object to the network layer for serialization.
    2.  **Client-Side (Deserialization):** Instantiated by a network protocol decoder (e.g., a Netty channel handler) when an incoming data buffer is identified as an entity update packet. The static `deserialize` method acts as the factory in this scenario.

-   **Scope:** The lifetime of an EntityUpdate instance is extremely short. It is a transient object, designed to exist only for the duration of a single transaction through the network or game logic pipeline.

-   **Destruction:** The object becomes eligible for garbage collection immediately after its data has been consumed. On the client, this is after the component changes have been applied to the local entity. On the server, this is after the object has been serialized into a network buffer. There is no manual memory management or cleanup required.

## Internal State & Concurrency

-   **State:** The class is fully mutable. Its public fields are directly accessible and are populated during construction or deserialization. It is designed as a simple data holder and does not encapsulate or protect its internal state.

-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed to be created, processed, and discarded within the context of a single thread, such as a Netty I/O worker or the main game loop thread.

    **WARNING:** Sharing an EntityUpdate instance across multiple threads without external, explicit synchronization will result in undefined behavior and is a critical programming error.

## API Surface

The public API is dominated by static methods for serialization, deserialization, and validation, reinforcing its role as a data specification rather than an active object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static EntityUpdate | O(N) | Constructs an EntityUpdate instance by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the instance's state into the provided ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a safe, read-only check of the buffer to verify structural integrity before attempting a full deserialization. Critical for security and stability. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total number of bytes this message occupies in a buffer without performing a full deserialization. Used by decoders to skip messages. |
| computeSize() | int | O(N) | Calculates the number of bytes the current instance will occupy when serialized. Useful for pre-allocating buffers. |

*N = Total size of all component data within the update.*

## Integration Patterns

### Standard Usage

**Server-Side: Sending an Update**
The server's core logic constructs an EntityUpdate, populates it, and hands it to the network layer to be serialized and sent to a client.

```java
// Example: Server logic preparing an update
ComponentUpdate[] updates = getEntityComponentChanges(entityId);
ComponentUpdateType[] removed = getEntityComponentRemovals(entityId);

EntityUpdate entityUpdate = new EntityUpdate(entityId, removed, updates);

// The network layer will then serialize it
networkChannel.write(entityUpdate); // Internally calls entityUpdate.serialize(buffer)
```

**Client-Side: Receiving an Update**
The client's network decoder uses the static methods to validate and deserialize the incoming data, then passes the resulting object to the game thread for processing.

```java
// Example: Client network handler
ByteBuf incomingBuffer = ...;
if (EntityUpdate.validateStructure(incomingBuffer, 0).isValid()) {
    EntityUpdate update = EntityUpdate.deserialize(incomingBuffer, 0);
    
    // Dispatch to the main game thread to apply changes
    gameThread.submit(() -> entityManager.applyUpdate(update));
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not hold a reference to an EntityUpdate object to modify and send it again later. These objects are cheap to create and should be treated as single-use messages to avoid complex state management issues.
-   **Cross-Thread Modification:** Do not deserialize an EntityUpdate on a network thread and then pass the mutable object to a game thread that might read from it while the network thread is still processing. A defensive copy or a thread-safe dispatch queue should be used.
-   **Skipping Validation:** Never call `deserialize` on a buffer received from an external source without first calling `validateStructure`. Doing so exposes the system to potential crashes or exploits from malformed packets.

## Data Pipeline

The EntityUpdate class is a key payload in the entity synchronization data flow.

**Outbound Flow (Server)**
> Game State Change → **EntityUpdate (Instantiation)** → `serialize()` → Netty ByteBuf → Encoded Packet → Network

**Inbound Flow (Client)**
> Network → Raw Bytes → Netty ByteBuf → `validateStructure()` → `deserialize()` → **EntityUpdate (Instance)** → Game Logic (Apply Changes) → Local Entity State Updated


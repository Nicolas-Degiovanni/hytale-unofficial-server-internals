---
description: Architectural reference for CombatTextUpdate
---

# CombatTextUpdate

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class CombatTextUpdate {
```

## Architecture & Concepts
The CombatTextUpdate class is a Data Transfer Object (DTO) that defines the wire protocol for displaying floating combat text in the game client. This includes damage numbers, healing amounts, or status effect labels that appear in the 3D world during combat.

This class is a fundamental component of the network protocol layer. It does not contain any game logic. Its sole responsibility is to act as a strict contract for serializing combat text data into a byte stream on the server and deserializing it from a byte stream on the client. The implementation is highly optimized for network performance, employing a custom binary format with bit fields for nullability and variable-length integers for string sizes.

It operates at a low level, directly manipulating Netty ByteBuf objects. This design ensures minimal overhead during the critical process of packet encoding and decoding, which is essential for a real-time game.

### Lifecycle & Ownership
- **Creation:**
    - On the **server**, an instance is created by the game logic thread in response to a combat event (e.g., an entity taking damage).
    - On the **client**, an instance is created exclusively by the static *deserialize* method, which is invoked by the network layer's packet dispatcher when a corresponding packet is read from the network buffer.
- **Scope:** This object is extremely short-lived and transient. Its scope is confined to the immediate task of data transfer. On the server, it exists only long enough to be passed to a serializer. On the client, it exists only until its data has been consumed by the UI or rendering system.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been processed. There is no persistent storage or caching of CombatTextUpdate instances.

## Internal State & Concurrency
- **State:** The class is a simple, mutable data container. Its public fields, *hitAngleDeg* and *text*, can be modified directly after instantiation. However, once deserialized on the client, it should be treated as an immutable value object.
- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization mechanisms. It is designed to be created, populated, and serialized on a single game thread, or deserialized and read on a single network thread (e.g., a Netty event loop). Concurrent modification from multiple threads will lead to data corruption and unpredictable behavior. All synchronization must be handled externally.

## API Surface
The public contract is dominated by static methods for low-level buffer manipulation, defining its role as a self-contained serialization utility.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | CombatTextUpdate | O(N) | **[Client]** Constructs a new object by reading from a ByteBuf at a given offset. N is the length of the text string. |
| serialize(ByteBuf) | void | O(N) | **[Server]** Writes the object's state into the provided ByteBuf. N is the length of the text string. |
| computeBytesConsumed(ByteBuf, int) | int | O(1) | Calculates the total number of bytes this packet occupies in a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes required to serialize the current object state. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a lightweight check on a buffer to ensure it contains a structurally valid packet, preventing crashes from malformed data. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the core networking engine. A packet dispatcher identifies the packet type and invokes the static *deserialize* method. The resulting object is then typically passed into the game's event system.

```java
// Executed by the client's network packet handler
// buf contains the raw network data for this packet
CombatTextUpdate update = CombatTextUpdate.deserialize(buf, 0);

// The update object is then passed to another system for processing
gameEventHandler.handleCombatText(update);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Client:** Do not use *new CombatTextUpdate()* on the client side for network-related code. Always use the static *deserialize* method to ensure the object correctly represents the data received from the server.
- **Object Re-use:** Do not attempt to pool or re-use CombatTextUpdate instances. They are lightweight objects, and the overhead of managing a pool outweighs the cost of garbage collecting these short-lived DTOs.
- **State Modification After Deserialization:** Modifying a deserialized CombatTextUpdate object is dangerous. It represents a point-in-time event from the server and should be treated as immutable once received to prevent state desynchronization bugs.

## Data Pipeline
The flow of data for this object is unidirectional from server to client.

> **Server Flow:**
> Combat Event -> Game Logic creates **CombatTextUpdate** -> Serializer calls *serialize()* -> Netty writes ByteBuf to Network Socket

> **Client Flow:**
> Netty reads from Network Socket into ByteBuf -> Packet Dispatcher calls *deserialize()* -> **CombatTextUpdate** object created -> Event Bus -> UI/Rendering System displays text


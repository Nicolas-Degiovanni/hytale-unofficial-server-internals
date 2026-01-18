---
description: Architectural reference for Selector
---

# Selector

**Package:** com.hypixel.hytale.protocol
**Type:** Utility / Polymorphic Base Class

## Definition
```java
// Signature
public abstract class Selector {
```

## Architecture & Concepts

The Selector class is a foundational component of the Hytale network protocol, serving as the abstract base for a polymorphic data structure. Its primary role is to orchestrate the serialization and deserialization of various targeting and area-of-effect mechanisms used in gameplay. These mechanisms, such as a circular area of effect or a raycast, are represented by concrete subclasses like AOECircleSelector and RaycastSelector.

Architecturally, Selector implements a variant of the **Type-Length-Value (TLV)** encoding pattern. An integer type identifier, encoded as a VarInt, is prepended to the data stream. This identifier acts as a dispatch key, allowing the static factory method, deserialize, to instantiate the correct concrete subclass to parse the subsequent data.

This design provides critical protocol extensibility. New targeting behaviors can be introduced by adding new subclasses and registering a unique type identifier, without requiring changes to the core packet-handling logic. The class provides a unified contract for encoding, decoding, size calculation, and validation, ensuring that all selector types are handled consistently by the protocol layer.

### Lifecycle & Ownership
-   **Creation:** Selector instances are not created via a public constructor. They are materialized exclusively through the static `Selector.deserialize` factory method. This occurs deep within the network protocol stack when a raw ByteBuf from an incoming packet is being parsed into a structured game object.
-   **Scope:** These are **transient, short-lived Data Transfer Objects (DTOs)**. Their lifecycle is typically bound to the processing of a single network packet. They exist only to carry data from the network into the game logic.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as the game event or packet handler that consumed them completes its execution. There are no manual cleanup or disposal requirements.

## Internal State & Concurrency
-   **State:** The abstract Selector class is stateless. All state is encapsulated within its concrete subclasses (e.g., radius for AOECircleSelector). By design, these objects should be treated as **immutable** after deserialization. They represent a snapshot of data transmitted over the network at a specific point in time.
-   **Thread Safety:** The static methods are inherently thread-safe as they are pure functions operating on a given ByteBuf without modifying any shared static state. Instance methods are safe under the assumption that a single Selector object is not mutated or accessed concurrently after its creation.

    **Warning:** The underlying Netty ByteBuf is not guaranteed to be thread-safe for concurrent access. Consumers must ensure that the buffer passed to Selector methods is accessed from only one thread at a time or is properly synchronized externally.

## API Surface

The public API is designed for interaction with the network protocol layer, focusing on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static Selector | O(N) | Factory method. Reads a type ID from the buffer and delegates to the corresponding subclass to construct a new Selector instance. Throws ProtocolException for unknown type IDs. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total number of bytes a Selector object occupies in the buffer without allocating a new object. Essential for skipping data or pre-calculating packet lengths. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Peeks at the type ID and delegates to the appropriate subclass validator. Used to verify data integrity before full deserialization, preventing malformed packet exploits. |
| getTypeId() | int | O(1) | Returns the unique integer identifier for the concrete subclass instance. |
| serializeWithTypeId(buf) | int | O(N) | The primary serialization method. Writes the type ID followed by the subclass-specific payload to the buffer. Returns the number of bytes written. |
| computeSizeWithTypeId() | int | O(1) | Calculates the total serialized size of the instance, including its type ID. |

## Integration Patterns

### Standard Usage

The primary use case is deserializing a Selector from a network buffer during packet processing. The `computeBytesConsumed` method should be used to correctly advance the buffer's read position.

```java
// In a packet handler, processing an incoming ByteBuf
int currentOffset = ...;
ValidationResult result = Selector.validateStructure(packetBuffer, currentOffset);
if (!result.isSuccess()) {
    // Handle validation failure; disconnect client
    return;
}

Selector selector = Selector.deserialize(packetBuffer, currentOffset);
int bytesRead = Selector.computeBytesConsumed(packetBuffer, currentOffset);
currentOffset += bytesRead;

// Pass the concrete selector object to the game logic
gameSystem.applyEffect(selector, targetEntity);
```

### Anti-Patterns (Do NOT do this)
-   **Buffer Index Mismanagement:** Failing to use `computeBytesConsumed` to advance the buffer's read index after a `deserialize` call is a critical error. This will cause subsequent reads from the buffer to parse incorrect data, leading to protocol desynchronization and likely a connection drop.
-   **Ignoring Validation:** On the server, directly calling `deserialize` on a buffer from an untrusted client without first calling `validateStructure` is a security risk. A malformed payload could trigger a ProtocolException that crashes the packet processing thread or, in worse cases, exploit logic bugs.
-   **Manual Subclass Instantiation:** While technically possible to use `new AOECircleSelector()`, these objects are intended to be created by the protocol layer. Manually constructing them for game logic can tightly couple the logic to a specific selector type, defeating the purpose of the polymorphic design.

## Data Pipeline

The Selector class is a key transformation step in the inbound network data pipeline, converting a raw byte stream into a structured, usable game object.

> Flow:
> Raw TCP Stream -> Netty Channel Handler -> Packet Frame Decoder -> ByteBuf with Selector Data -> **Selector.deserialize()** -> Concrete Selector Object (e.g., RaycastSelector) -> Game Event Bus -> Gameplay System (e.g., CombatHandler)


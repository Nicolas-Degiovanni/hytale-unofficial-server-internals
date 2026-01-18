---
description: Architectural reference for RoofConnectedBlockRuleSet
---

# RoofConnectedBlockRuleSet

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class RoofConnectedBlockRuleSet {
```

## Architecture & Concepts
The RoofConnectedBlockRuleSet is a Data Transfer Object (DTO) designed for high-performance network serialization and deserialization. It is not an active system component but rather a passive data container that represents the logical rules for how "roof" type blocks should visually connect to their neighbors. This data is critical for the client-side rendering engine to select the appropriate 3D model or texture variant, creating seamless and complex structures.

Architecturally, this class embodies a common pattern in high-performance binary protocols: a hybrid fixed-and-variable layout. The serialized structure consists of:
1.  A **fixed-size header** (21 bytes) containing primitive values and offsets.
2.  A **nullability bitmask** (the first byte) to efficiently track the presence of optional, variable-sized fields. This avoids the overhead of writing explicit presence flags or terminators for each field.
3.  A **variable-size data region** appended after the header, containing the actual data for complex types like strings or nested objects. The header contains integer offsets pointing to the start of each data element within this region.

This design allows for extremely fast reads of the fixed data and provides a direct addressing mechanism for variable data, enabling parsers to skip optional fields they are not interested in without reading their content.

## Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the network layer through the static factory method **deserialize**. This occurs when a network packet containing roof block rule data is read from a Netty ByteBuf. On the server, instances are created via standard constructors before being passed to a serializer.
-   **Scope:** The object's lifetime is ephemeral. It is scoped to the processing of a single network packet or a single game state update. It exists to transfer data from a raw byte stream into a structured, in-memory representation that other game systems can consume.
-   **Destruction:** The object is managed by the Java Garbage Collector. Once all references are dropped—typically after the relevant game systems have processed its data—it becomes eligible for garbage collection. No manual resource management is required.

## Internal State & Concurrency
-   **State:** The internal state is fully **mutable**. All data fields are public, prioritizing raw performance and ease of access within the protocol layer over encapsulation. The object is intended to be populated once during deserialization and then treated as a read-only value object by the rest of the application.

-   **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single-threaded context, such as a Netty event loop or the main game thread. Concurrent modification from multiple threads will result in data corruption and undefined behavior. Any multi-threaded access must be protected by external synchronization mechanisms.

## API Surface
The public contract is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a data codec.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | RoofConnectedBlockRuleSet | O(N) | **[Primary Constructor]** Reads from a ByteBuf at a given offset and constructs a new instance. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a structural integrity check on the data in a buffer without performing a full deserialization. Critical for security and preventing parsing errors. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total number of bytes the serialized object occupies in a buffer, allowing a parser to efficiently skip over it. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. Used for pre-allocating buffers. |

## Integration Patterns

### Standard Usage
The class is intended to be used by a higher-level packet handler that decodes a network buffer into game-specific data structures.

```java
// Example within a hypothetical PacketProcessor
// The buffer contains a stream of data from the network.

// Before deserializing, validate the structure to prevent errors.
ValidationResult result = RoofConnectedBlockRuleSet.validateStructure(networkBuffer, offset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid RoofConnectedBlockRuleSet: " + result.error());
}

// Deserialize into a usable object.
RoofConnectedBlockRuleSet rules = RoofConnectedBlockRuleSet.deserialize(networkBuffer, offset);

// Pass the immutable data to the relevant game system.
world.getBlockDataManager().registerRules(rules);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation After Deserialization:** Do not modify the public fields of an instance after it has been deserialized and passed to other systems. This breaks the implicit contract that the object is a stable representation of a network message and can lead to severe state desynchronization.
-   **Ignoring Validation:** Never call **deserialize** on a buffer received from an untrusted source (like a game client or public server) without first calling **validateStructure**. Bypassing validation can expose the application to buffer overflows or denial-of-service attacks via malformed packets.
-   **Shared Mutable Access:** Do not pass a reference to an instance to multiple threads. If data needs to be shared, either create a deep copy using the **clone** method for each thread or implement an external locking strategy.

## Data Pipeline
The primary flow involves the transformation of raw bytes from the network into a structured object that the game engine can interpret.

> Flow (Client-Side):
> Network Byte Stream -> Netty I/O Thread -> ByteBuf -> **RoofConnectedBlockRuleSet.deserialize()** -> In-Memory RoofConnectedBlockRuleSet Object -> Game Logic / Rendering Engine


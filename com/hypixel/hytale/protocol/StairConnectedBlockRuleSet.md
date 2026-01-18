---
description: Architectural reference for StairConnectedBlockRuleSet
---

# StairConnectedBlockRuleSet

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class StairConnectedBlockRuleSet {
```

## Architecture & Concepts
The StairConnectedBlockRuleSet is a fundamental data structure within the Hytale protocol layer. It serves as a self-contained "message" or "record" that defines the rendering rules for stair blocks. This class is not an active component with behavior; rather, it is a passive container for configuration data transmitted from the server to the client.

Its primary role is to inform the client-side rendering engine how to implement "connected block" logic specifically for stairs. Based on the spatial relationship with adjacent blocks, a stair block can take on several forms: a straight piece, an inner corner, an outer corner, etc. This class provides the explicit block ID mappings for each of these configurations.

The `materialName` field suggests that these rulesets are grouped by material type (e.g., "wood_stairs", "stone_stairs"), allowing different block sets to share connection behaviors. The client consumes these objects upon joining a world to populate its internal block definition registries.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Deserialization:** The static `deserialize` factory method is called by the network protocol handler when a corresponding packet is read from a Netty ByteBuf. This is the most common creation path on the client.
    2.  **Direct Instantiation:** On the server, instances are created programmatically via its constructor to define game content and configuration before being serialized and sent to clients.
- **Scope:** This is a transient object. Its lifetime is typically short. On the client, an instance exists from the moment it is deserialized until its data has been consumed and integrated into a more permanent, engine-side data structure (like a block property map or registry). It is then eligible for garbage collection.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no manual resource management or `close` methods. Ownership is relinquished once the network handler or content loader finishes processing it.

## Internal State & Concurrency
- **State:** The StairConnectedBlockRuleSet is a mutable data container. All of its fields are public and can be modified directly after creation. It holds no caches and its state is a direct representation of the serialized data.

- **Thread Safety:** This class is **not thread-safe**. The public, mutable fields and lack of internal synchronization make it inherently unsafe for concurrent modification. It is designed to be processed within a single thread, such as a Netty I/O worker thread or the main game thread.

    **WARNING:** Sharing an instance of this class across multiple threads without external locking or synchronization will lead to race conditions and unpredictable behavior.

## API Surface
The public API is focused on serialization, deserialization, and data validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static StairConnectedBlockRuleSet | O(N) | Constructs an object by reading from a ByteBuf. N is the length of the materialName string. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf according to the protocol specification. N is the length of the materialName string. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. N is the length of the materialName string. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Calculates the size of a serialized object directly from a buffer without full deserialization. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a lightweight check on a buffer to ensure it contains a structurally valid object. Does not perform a full deserialization. |
| clone() | StairConnectedBlockRuleSet | O(N) | Creates a shallow copy of the object. N is the length of the materialName string. |

## Integration Patterns

### Standard Usage
This object is almost exclusively handled by the protocol layer. A typical use case involves reading it from a buffer as part of a larger packet handling process.

```java
// Executed within a Netty channel handler or similar context
ByteBuf packetData = ...;

// Before deserializing, validate the structure to prevent errors
ValidationResult result = StairConnectedBlockRuleSet.validateStructure(packetData, 0);
if (!result.isOk()) {
    throw new ProtocolException("Invalid StairConnectedBlockRuleSet: " + result.getReason());
}

// Deserialize the object to be passed to the game logic
StairConnectedBlockRuleSet rules = StairConnectedBlockRuleSet.deserialize(packetData, 0);
gameContentRegistry.registerStairRules(rules);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not pass an instance to another thread for modification while the original thread still holds a reference. If data must be shared, create a deep copy using the copy constructor or `clone` method.
- **Deserializing Untrusted Data:** Never call `deserialize` on a buffer received from the network without first calling `validateStructure`. Failure to do so can result in `ProtocolException` or, in worse cases, excessive memory allocation if a malicious client specifies an extremely large string length.
- **Manual Serialization:** Do not attempt to write the fields to a buffer manually. The `serialize` method correctly handles the `nullBits` bitfield, which is critical for the protocol to function. Manually writing the fields will produce an invalid payload.

## Data Pipeline
The StairConnectedBlockRuleSet acts as a data record that flows from server configuration to the client's rendering system.

> **Server Flow:**
> Game Asset Files (e.g., JSON) -> Server Content Loader -> **StairConnectedBlockRuleSet** (In-Memory) -> `serialize()` -> Network Packet (ByteBuf) -> Client

> **Client Flow:**
> Network Packet (ByteBuf) -> Protocol Decoder -> `deserialize()` -> **StairConnectedBlockRuleSet** (In-Memory) -> Block Definition Registry -> Rendering Engine Logic


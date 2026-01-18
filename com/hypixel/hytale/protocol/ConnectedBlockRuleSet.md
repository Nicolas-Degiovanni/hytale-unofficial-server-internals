---
description: Architectural reference for ConnectedBlockRuleSet
---

# ConnectedBlockRuleSet

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ConnectedBlockRuleSet {
```

## Architecture & Concepts
The ConnectedBlockRuleSet is a data transfer object (DTO) that defines the serialization and deserialization logic for a set of rules governing how certain game blocks connect visually. It is a fundamental component of the Hytale network protocol, acting as a structured container for data transmitted between the server and client.

This class is not a service or manager; it is a pure data structure. Its primary architectural role is to provide a strict, high-performance binary representation for complex, variable-size game data. The serialization format is custom, employing a bitmask for nullable fields and relative offsets for variable-length child objects. This design minimizes payload size and allows for efficient, direct-from-buffer parsing without intermediate allocations, which is critical for the performance of the network layer.

It operates exclusively within the protocol layer, translating raw byte streams from a Netty ByteBuf into a usable Java object, and vice-versa.

### Lifecycle & Ownership
- **Creation:** Instances are created under two primary conditions:
    1.  By the network protocol layer when a packet is received, via the static `deserialize` method.
    2.  By server-side game logic or asset loading systems that need to construct a ruleset to be sent to a client.
- **Scope:** The lifetime of a ConnectedBlockRuleSet object is extremely short. It is designed to be transient, existing only for the duration of a single network event or game logic operation.
- **Destruction:** The object is eligible for garbage collection as soon as the network packet has been fully processed or the serialization operation is complete. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The object's state is fully mutable. Its public fields can be directly accessed and modified after construction. It is essentially a container for the `type`, `stair`, and `roof` properties. It holds no cached data or derived state.
- **Thread Safety:** This class is **not thread-safe**. It contains no synchronization mechanisms. Instances must be confined to a single thread. In a typical Hytale architecture, this would be a Netty I/O worker thread during deserialization or the main game thread during processing. Concurrent modification will lead to data corruption and unpredictable behavior.

## API Surface
The public contract is focused entirely on binary serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ConnectedBlockRuleSet | O(N) | **[Static]** Constructs an object by reading from a ByteBuf at a given offset. N is the size of the data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf. N is the size of the data. |
| computeSize() | int | O(1) | Calculates the total byte size the object will occupy when serialized. |
| computeBytesConsumed(buf, offset) | int | O(N) | **[Static]** Calculates the size of a serialized object directly from a buffer without full deserialization. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | **[Static]** Performs critical bounds and offset checks on a buffer before deserialization. |
| clone() | ConnectedBlockRuleSet | O(1) | Creates a shallow copy of the object and deep copies of its child rule sets. |

## Integration Patterns

### Standard Usage
This class should only be used by the network protocol handlers or systems that directly prepare data for network transmission. The typical flow involves validating the buffer segment before attempting to deserialize.

```java
// Deserializing from a network buffer
ByteBuf networkBuffer = ...;
int dataOffset = ...;

ValidationResult result = ConnectedBlockRuleSet.validateStructure(networkBuffer, dataOffset);
if (result.isValid()) {
    ConnectedBlockRuleSet ruleSet = ConnectedBlockRuleSet.deserialize(networkBuffer, dataOffset);
    // ... process the ruleSet in the game logic
} else {
    // Handle or log the malformed packet
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Deserialization:** Do not use `new ConnectedBlockRuleSet()` and then manually populate fields from a buffer. Always use the static `deserialize` method, as it correctly handles the complex binary layout including null bitfields and variable offsets.
- **Ignoring Validation:** Never call `deserialize` on a buffer received from an untrusted source (like a game client) without first calling `validateStructure`. Failure to do so exposes the server to buffer-over-read vulnerabilities and denial-of-service attacks from malformed packets.
- **Long-Term Storage:** Do not hold references to ConnectedBlockRuleSet instances for extended periods. They are transient DTOs, not components of the persistent game state. Retaining them can lead to increased memory pressure.

## Data Pipeline
ConnectedBlockRuleSet serves as a model for data in transit. It represents a single transformation step in the data flow between the network and the game engine.

> **Inbound Flow (Server):**
> Netty Channel -> ByteBuf -> Protocol Decoder -> **ConnectedBlockRuleSet.validateStructure** -> **ConnectedBlockRuleSet.deserialize** -> Game Event Bus -> World Logic

> **Outbound Flow (Server):**
> World Logic -> **new ConnectedBlockRuleSet()** -> Protocol Encoder -> **ConnectedBlockRuleSet.serialize(buf)** -> ByteBuf -> Netty Channel


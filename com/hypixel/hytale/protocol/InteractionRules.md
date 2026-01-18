---
description: Architectural reference for InteractionRules
---

# InteractionRules

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class InteractionRules {
```

## Architecture & Concepts
InteractionRules is a Data Transfer Object (DTO) that defines the behavioral rules governing how game entities can interact with each other. It is not a service or manager, but a pure data container that represents a specific configuration of interaction permissions.

Architecturally, this class serves as a concrete schema for a highly optimized binary serialization format used within the Hytale network protocol. Its primary function is to be encoded into or decoded from a raw stream of bytes (a Netty ByteBuf) sent between the client and server.

The serialization format is custom-designed for efficiency:
1.  **Fixed Header:** A 33-byte block contains a bitmask for nullable array fields and fixed-size integer fields.
2.  **Nullable Field Optimization:** The initial byte acts as a bitmask, indicating which of the four variable-sized arrays are present in the payload. If an array is null, its data is omitted entirely, saving bandwidth.
3.  **Variable Data Block:** Following the fixed header, a variable-sized block contains the actual array data. The header contains integer offsets pointing to the start of each array's data within this block. This allows for flexible data layout and avoids parsing data that is not needed.

This structure makes InteractionRules a critical component for communicating complex, state-dependent game logic (e.g., "a player holding a special key can interact with a locked door") in a compact and performant manner.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the static factory method **InteractionRules.deserialize** when a network packet handler processes an incoming ByteBuf. On the sending side, instances are created via the constructor to be serialized into an outgoing packet.
- **Scope:** The lifetime of an InteractionRules object is extremely short and tied to the scope of a single network packet's processing cycle. It is a transient, value-like object.
- **Destruction:** The object is eligible for garbage collection immediately after the consuming system (e.g., an action processing system) has read its state. No manual resource management is required.

## Internal State & Concurrency
- **State:** The object's state is fully mutable. All fields are public, allowing for direct modification. This design is intentional for a DTO, prioritizing high-performance serialization and direct access over encapsulation. The object holds no derived or cached state; it is a direct representation of the serialized data.
- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game update loop. Sharing an instance across multiple threads without external locking mechanisms will lead to race conditions and undefined behavior.

## API Surface
The primary contract is defined by the static serialization and validation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static InteractionRules | O(N) | Constructs an InteractionRules object by decoding a binary payload from a buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided buffer according to the custom binary format. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check on a buffer to verify structural integrity without full deserialization. Crucial for network security and stability. |
| computeBytesConsumed(ByteBuf, int) | static int | O(1) | Calculates the total size of a serialized object within a buffer by reading its header and array lengths. |
| clone() | InteractionRules | O(N) | Creates a deep copy of the object, including new instances of all internal arrays. |

*N = total number of elements across all InteractionType arrays.*

## Integration Patterns

### Standard Usage
The canonical use case is within a network packet handler to decode interaction data sent from the server.

```java
// Inside a packet handler receiving a ByteBuf
public void handlePacket(ByteBuf buffer) {
    // Validate before deserializing to prevent protocol errors
    ValidationResult result = InteractionRules.validateStructure(buffer, buffer.readerIndex());
    if (!result.isOk()) {
        // Disconnect client or log error
        return;
    }

    InteractionRules rules = InteractionRules.deserialize(buffer, buffer.readerIndex());
    int bytesConsumed = InteractionRules.computeBytesConsumed(buffer, buffer.readerIndex());
    buffer.skipBytes(bytesConsumed);

    // Pass the 'rules' object to the game's action or physics system
    game.getInteractionSystem().applyRules(rules);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not hold references to InteractionRules objects beyond the immediate scope of their use. They represent a point-in-time snapshot of rules and should be re-acquired or deserialized as needed.
- **Concurrent Modification:** Never access or modify an InteractionRules instance from multiple threads. If rules need to be shared, create a deep copy for each thread using the clone method.
- **Ignoring Validation:** Bypassing validateStructure before calling deserialize on untrusted network data exposes the server to potential ProtocolException crashes or buffer overflow vulnerabilities.

## Data Pipeline
InteractionRules acts as a data marshalling object between the network layer and the core game logic.

> **Flow (Receiving):**
> Raw Network ByteBuf -> **InteractionRules.validateStructure** -> **InteractionRules.deserialize** -> Game Interaction System -> Gameplay Decision

> **Flow (Sending):**
> Game State Change -> `new InteractionRules(...)` -> **InteractionRules.serialize** -> Raw Network ByteBuf -> Client/Server
---


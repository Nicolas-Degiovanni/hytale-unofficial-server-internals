---
description: Architectural reference for EffectConditionInteraction
---

# EffectConditionInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Data Model

## Definition
```java
// Signature
public class EffectConditionInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The EffectConditionInteraction class is a specialized data model representing a conditional node within the game's interaction state machine. It is not a service or manager, but rather a Plain Old Java Object (POJO) designed for efficient network serialization and deserialization.

Architecturally, it serves as a conditional gate. Its primary purpose is to check whether a target entity possesses a specific set of status effects. Based on this check, the interaction state machine transitions to one of two subsequent states, defined by the inherited *next* and *failed* fields. This allows for creating complex, branching behaviors in game content—such as quests, scripted events, or dynamic NPC responses—without hard-coding logic into the engine.

The binary format of this class is highly optimized for network performance. It employs a hybrid layout consisting of a fixed-size block and a variable-size data block.

-   **Fixed Block (21 bytes):** Contains primitive types and enums that are always present. This allows for extremely fast, direct memory access.
-   **Variable Block Pointers (24 bytes):** A series of integer offsets within the fixed block that point to the location of variable-sized data (like arrays or nested objects) in the subsequent data block.
-   **Nullable Bit Field (1 byte):** A bitmask at the start of the payload that efficiently encodes the presence or absence of nullable fields. Each bit corresponds to a specific nullable field, avoiding the need for larger null markers.

This structure minimizes packet size and reduces CPU overhead during deserialization, which is critical for a high-performance game server.

### Lifecycle & Ownership
-   **Creation:** Instances are primarily created by the protocol layer when deserializing an incoming network packet via the static **deserialize** factory method. They can also be instantiated programmatically on the server by game logic before being serialized for transmission to a client.
-   **Scope:** An EffectConditionInteraction object is transient. Its lifetime is typically confined to the processing of a single network packet or a single tick of the game logic that evaluates it. It is not designed to be a long-lived, persistent entity.
-   **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection as soon as it is no longer referenced by the network buffer processor or the interaction state machine. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The class is fundamentally mutable. Its public fields can be directly accessed and modified after instantiation. This design prioritizes performance and ease of use within a single-threaded context over enforced immutability.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for use exclusively within a single thread, such as a Netty I/O thread or the main server game loop. Concurrent access from multiple threads without external synchronization will lead to race conditions, data corruption, and undefined behavior.

**WARNING:** Never share an instance of EffectConditionInteraction across threads. If state must be passed to another thread, create a deep copy using the **clone** method.

## API Surface
The primary contract of this class revolves around its static serialization and validation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | EffectConditionInteraction | O(N) | **Static.** Constructs an object by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state to a ByteBuf. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **Static.** Performs a structural integrity check on the binary data without full deserialization. Critical for security. |
| computeBytesConsumed(buf, offset) | int | O(N) | **Static.** Calculates the total size of a serialized object in a buffer. Used for skipping records. |
| clone() | EffectConditionInteraction | O(N) | Creates a deep copy of the object and its nested data structures. |

## Integration Patterns

### Standard Usage
The most common use case is deserializing the object from a network buffer as part of a larger packet-handling pipeline. The structure must be validated before deserialization to prevent server exceptions from malformed client data.

```java
// In a network handler receiving a ByteBuf
ByteBuf buffer = ...;
int offset = ...;

// 1. ALWAYS validate data from an untrusted source first.
ValidationResult result = EffectConditionInteraction.validateStructure(buffer, offset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid EffectConditionInteraction: " + result.error());
}

// 2. Deserialize into a usable object.
EffectConditionInteraction interaction = EffectConditionInteraction.deserialize(buffer, offset);

// 3. Pass the object to the game logic for processing.
interactionSystem.process(interaction);
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Never call **deserialize** on data from a remote source without first calling **validateStructure**. Bypassing validation exposes the server to denial-of-service attacks via malformed packets.
-   **Concurrent Modification:** Do not read or write to an instance from multiple threads. The object's state is not protected by locks and will become corrupt.
-   **State Reuse:** Do not reuse an object instance for a new network message without performing a deep copy via **clone**. Modifying a cached instance can lead to subtle and hard-to-trace bugs.

## Data Pipeline
EffectConditionInteraction functions as a data payload within the server's core logic pipeline. It is decoded from raw bytes and transformed into an actionable game logic instruction.

> Flow:
> Raw Network Bytes (ByteBuf) -> Protocol Decoder -> **EffectConditionInteraction.deserialize** -> Interaction State Machine -> Condition Evaluation -> State Transition (to *next* or *failed*)


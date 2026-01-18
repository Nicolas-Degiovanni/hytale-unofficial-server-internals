---
description: Architectural reference for BenchTierLevel
---

# BenchTierLevel

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class BenchTierLevel {
```

## Architecture & Concepts
The BenchTierLevel class is a Data Transfer Object (DTO) that represents a single tier of a crafting station's progression within the Hytale protocol. It is not a service or a manager; it is a passive data structure that strictly defines the wire format for crafting bench attributes.

This class acts as a fundamental building block for game state synchronization. When the server communicates changes or definitions related to crafting stations—such as their capabilities, upgrade requirements, or efficiency—it serializes this data into a BenchTierLevel structure. The client then deserializes the byte stream back into a BenchTierLevel object to update its local game state.

Its design is optimized for network performance, featuring a fixed-size block for primitive fields and a variable-size block for nested objects. This allows for efficient, predictable deserialization on the network thread.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the static `deserialize` method when the network layer processes an incoming packet. Game logic on the server may also instantiate this class directly to define new crafting bench behaviors before serializing them for clients.
- **Scope:** Extremely short-lived. An instance typically exists only for the duration of a single network packet's processing cycle or a single game logic update. It is a temporary container for data in transit.
- **Destruction:** The object is eligible for garbage collection as soon as the system that received it (e.g., a crafting manager) has consumed its data and no longer holds a reference. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Highly mutable. All data-holding fields are public and can be modified directly after instantiation. The class performs no internal caching and holds no references to engine services. It is a pure data container.
- **Thread Safety:** **This class is not thread-safe.** Its public, mutable fields make it inherently unsafe for concurrent modification or access from multiple threads. It is designed to be created, processed, and discarded within a single, synchronized context, such as a Netty I/O thread or the main game update loop.

**WARNING:** Do not share instances of BenchTierLevel across threads without explicit, external locking. Doing so will lead to race conditions and unpredictable game state corruption.

## API Surface
The primary contract of this class revolves around its serialization and validation capabilities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BenchTierLevel | O(N) | **[Static]** Constructs a new BenchTierLevel by reading from a ByteBuf at a given offset. N is the size of the variable data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf. |
| computeBytesConsumed(buf, offset) | int | O(N) | **[Static]** Calculates the total number of bytes this object occupies in a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes required to serialize this object. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | **[Static]** Performs a structural integrity check on the data in a buffer without creating an object. |
| clone() | BenchTierLevel | O(N) | Creates a deep copy of the object and its nested structures. |

## Integration Patterns

### Standard Usage
The class is intended to be used as part of the network deserialization pipeline. Game logic should retrieve the object, read its values to update the state of a more permanent game entity, and then discard the reference.

```java
// Executed on a network thread or game thread
// The ByteBuf contains incoming packet data
int offset = ...; // Start of the BenchTierLevel data

if (BenchTierLevel.validateStructure(packetBuffer, offset).isValid()) {
    BenchTierLevel tierData = BenchTierLevel.deserialize(packetBuffer, offset);

    // Apply the data to the actual game object
    CraftingBench bench = getCraftingBenchById(...);
    bench.setTierProperties(tierData);
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not hold onto a BenchTierLevel instance and modify it over time. It is designed to represent a point-in-time snapshot of state from the network. Reusing instances can lead to confusing and buggy behavior.
- **Concurrent Access:** Never pass a BenchTierLevel instance to another thread for processing without deep-copying it via the `clone` method first.
- **Manual Serialization:** Avoid manually writing the fields to a buffer. Always use the `serialize` method to ensure compliance with the wire format, especially regarding the nullable bit field.

## Data Pipeline
BenchTierLevel is not a processing stage but rather the data payload itself. It represents deserialized, structured information ready for consumption by higher-level game systems.

> Flow:
> Raw ByteBuf from Network -> **BenchTierLevel.deserialize** -> BenchTierLevel Instance -> Crafting System Logic -> Game State Update


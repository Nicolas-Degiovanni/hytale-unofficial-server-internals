---
description: Architectural reference for BrushAxis
---

# BrushAxis

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Utility / Value Type

## Definition
```java
// Signature
public enum BrushAxis {
```

## Architecture & Concepts
BrushAxis is a type-safe enumeration that represents the coordinate axes for in-game builder tools. It serves as a critical data contract within the network protocol, ensuring that both the client and server have a consistent and unambiguous understanding of tool orientation.

This enum's primary function is to translate a high-level concept—a spatial axis—into a low-level, network-safe integer representation. By encapsulating this logic, it prevents the propagation of "magic numbers" throughout the codebase and provides a single, authoritative source for axis definitions. It is a fundamental building block for serializing and deserializing any packet related to builder tool operations.

### Lifecycle & Ownership
- **Creation:** Instances of this enum (None, Auto, X, Y, Z) are constructed and managed exclusively by the Java Virtual Machine during class loading. User code does not, and cannot, instantiate this type directly.
- **Scope:** Application-wide static constants. These instances persist for the entire lifetime of the JVM process.
- **Destruction:** Instances are reclaimed by the JVM upon application shutdown. There is no manual memory management or destruction required.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final integer value that is assigned at creation and can never be changed. The static VALUES array is also final, providing a stable cache for lookup operations.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature and the JVM's guarantees for enum initialization, BrushAxis can be safely accessed and used across any number of threads without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | BrushAxis | O(1) | A static factory method to deserialize an integer from a network packet into its corresponding BrushAxis constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is almost exclusively used within packet handlers and serialization logic to encode and decode builder tool data.

```java
// Example: Deserializing a brush axis from a network buffer
int axisValue = buffer.readVarInt();
BrushAxis axis = BrushAxis.fromValue(axisValue);

// Now 'axis' can be safely used in game logic
builderTool.setAxis(axis);
```

### Anti-Patterns (Do NOT do this)
- **Unhandled Exceptions:** The fromValue method will throw a runtime ProtocolException for an invalid integer. Failure to handle this exception in a network packet handler can lead to a channel disconnect and client termination. Always deserialize within a try-catch block.
- **Relying on Ordinals:** Do not use the built-in `ordinal()` method for serialization. The explicit `value` field is used to guarantee that the network representation remains stable even if the declaration order of the enum constants changes in the future. The current implementation correctly follows this best practice.

## Data Pipeline
BrushAxis acts as a data model, not a processing stage. It represents data at rest within the pipeline.

**Serialization (Outgoing Packet)**
> Flow:
> Game State (Builder Tool) -> **BrushAxis.getValue()** -> Integer written to Network Buffer

**Deserialization (Incoming Packet)**
> Flow:
> Integer read from Network Buffer -> **BrushAxis.fromValue()** -> BrushAxis instance in Game Logic


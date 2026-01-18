---
description: Architectural reference for LoopOption
---

# LoopOption

**Package:** com.hypixel.hytale.protocol
**Type:** Value Object Enum

## Definition
```java
// Signature
public enum LoopOption {
```

## Architecture & Concepts
The **LoopOption** enum is a foundational component of the Hytale network protocol. It provides a type-safe, compile-time constant representation for animation or sound looping behaviors. Its primary architectural function is to serve as a bridge between high-level game logic (e.g., "this animation should loop") and the low-level network layer, which requires a compact, serializable format.

By mapping symbolic names like **Loop** and **PlayOnce** to fixed integer values, the system ensures efficient data transmission. Sending a single byte integer is significantly more performant than sending a full string representation. The static factory method **fromValue** is the designated entry point for deserialization, converting raw integer data from a network stream back into a safe, managed object. This pattern is critical for protocol robustness, preventing invalid data from corrupting game state.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) during class loading. They are managed singletons, and developers should never attempt to create instances manually.
- **Scope:** Application-wide. The **LoopOption** constants exist for the entire lifetime of the running application.
- **Destruction:** The constants are garbage collected only when the JVM itself shuts down.

## Internal State & Concurrency
- **State:** **LoopOption** is strictly immutable. Its internal integer value is a final field assigned at creation time. Its state cannot be modified post-initialization.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutability and the JVM's management of enum instances, it can be safely accessed and shared across any number of threads without requiring external synchronization or locks.

## API Surface
The public contract of **LoopOption** is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value for network serialization. |
| fromValue(int value) | LoopOption | O(1) | Deserializes an integer into a LoopOption constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is within network packet decoders or data model hydration logic, where an integer must be safely converted back into its enum representation.

```java
// Deserializing a loop behavior from a network buffer
int loopTypeInt = buffer.readVarInt();
try {
    LoopOption option = LoopOption.fromValue(loopTypeInt);
    animationComponent.setLoopOption(option);
} catch (ProtocolException e) {
    // Handle malformed packet, e.g., disconnect client
    log.error("Received invalid LoopOption value: " + loopTypeInt);
}
```

### Anti-Patterns (Do NOT do this)
- **Comparison by Integer Value:** Never compare an enum instance by its underlying integer. This breaks type safety and creates brittle code.
  - **BAD:** `if (option.getValue() == 1)`
  - **GOOD:** `if (option == LoopOption.Loop)`
- **Reliance on Ordinal:** Do not use the built-in `ordinal()` method for serialization. It is highly fragile and will break if the declaration order of the enum constants changes. This class correctly provides a stable `getValue()` method for this purpose.
- **Ignoring Exceptions:** The **fromValue** method is a critical validation point. Failure to catch the **ProtocolException** it can throw will result in unhandled exceptions that can crash server or client threads when processing malformed or malicious network data.

## Data Pipeline
**LoopOption** does not process data itself; it *is* the data. It represents a state that is serialized for transport and deserialized upon receipt.

> **Serialization Flow:**
> Game State (AnimationComponent with **LoopOption.Loop**) -> Protocol Serializer -> `getValue()` -> Network Packet (contains integer `1`)

> **Deserialization Flow:**
> Network Packet (contains integer `1`) -> Protocol Deserializer -> **LoopOption.fromValue(1)** -> Game State (AnimationComponent with **LoopOption.Loop**)


---
description: Architectural reference for ShadingMode
---

# ShadingMode

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum ShadingMode {
```

## Architecture & Concepts
The ShadingMode enum defines a constrained, canonical set of rendering styles applicable to models, blocks, and other visual elements within the game engine. Its primary architectural role is to serve as a data contract for the network protocol, translating between a human-readable constant (e.g., Fullbright) and a low-level integer representation suitable for network serialization.

By replacing ambiguous "magic numbers" with a type-safe enumeration, ShadingMode prevents a class of common bugs related to rendering state. It ensures that both the client and server have an identical and validated understanding of how an object should be shaded, forming a critical part of the rendering data pipeline. The inclusion of the static factory method fromValue centralizes the logic for deserialization and validation, enforcing protocol correctness at the boundary.

### Lifecycle & Ownership
- **Creation:** All enum constants (Standard, Flat, etc.) are instantiated automatically by the Java Virtual Machine (JVM) during the class-loading phase. This process is guaranteed to happen only once, before any code can access the enum.
- **Scope:** Application-wide. The ShadingMode constants are effectively static singletons that persist for the entire lifetime of the application.
- **Destruction:** The enum instances are reclaimed by the JVM only during a full application shutdown. There is no manual lifecycle management.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a final integer value that is set at compile time and cannot be altered. The pre-computed VALUES array is also static and final.
- **Thread Safety:** **Inherently thread-safe**. Due to its complete immutability, ShadingMode can be safely accessed, passed, and read by any number of threads simultaneously without requiring locks or any other synchronization primitives.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value for network serialization. |
| fromValue(int value) | ShadingMode | O(1) | Deserializes an integer into a ShadingMode constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is deserializing a value from a network buffer and passing it to the rendering system. The fromValue method is the designated entry point for this.

```java
// Example: Reading a model's shading mode from a network packet
int modeId = networkBuffer.readVarInt();
ShadingMode shading = ShadingMode.fromValue(modeId);

// The renderer can now safely use the enum
renderingEngine.applyShading(shading);
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not rely on the `ordinal()` method for serialization. The integer value is explicitly defined for protocol stability, whereas the ordinal can change if the enum declaration order is modified.
- **Logic with Raw Integers:** Avoid comparing against the raw integer value in game logic. This defeats the purpose of a type-safe enum and re-introduces magic numbers. Always compare the enum constants directly.
    - **BAD:** `if (shading.getValue() == 2) { // logic for fullbright }`
    - **GOOD:** `if (shading == ShadingMode.Fullbright) { // logic for fullbright }`
- **Ignoring Exceptions:** The fromValue method is a critical validation point. Failure to catch the ProtocolException can result in a corrupted game state or client disconnect if a malicious or malformed packet is received.

## Data Pipeline
ShadingMode acts as a data representation that is serialized and deserialized at the protocol boundaries. It does not process data itself but ensures data integrity as it flows from the server's game state to the client's renderer.

> Flow:
> Server Game State -> **ShadingMode.getValue()** -> Network Packet Serialization -> Client Packet Deserialization -> **ShadingMode.fromValue(intValue)** -> Client Rendering Engine


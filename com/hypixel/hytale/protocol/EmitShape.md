---
description: Architectural reference for EmitShape
---

# EmitShape

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum EmitShape {
```

## Architecture & Concepts
The EmitShape enum provides a type-safe, integer-backed representation for a fixed set of geometric shapes. Its primary function is to serve as a data contract within the network protocol, ensuring that both the client and server have a consistent and unambiguous understanding of shape identifiers.

This class is a foundational element of the protocol layer. It translates a raw integer, which is what is serialized and transmitted over the network, into a robust, self-documenting Java object. The static factory method, fromValue, acts as the primary deserialization entry point, while the getValue method facilitates serialization.

The inclusion of a custom ProtocolException for invalid values tightly couples this enum to the protocol's error handling and validation pipeline. An invalid shape value is considered a protocol violation, not a general application error.

### Lifecycle & Ownership
- **Creation:** Instances are created by the Java Virtual Machine (JVM) during class loading. The set of instances, Sphere and Cube, is fixed and cannot be extended at runtime.
- **Scope:** Application-wide. These constants are static and exist for the entire lifetime of the application.
- **Destruction:** Cleaned up by the JVM during class unloading when the application shuts down.

## Internal State & Concurrency
- **State:** Inherently immutable. Each enum constant is a singleton instance with a final *value* field. Its state cannot be changed after creation. The static VALUES array is also final and populated only once.
- **Thread Safety:** Fully thread-safe. As immutable singletons, instances can be safely accessed and shared across any number of threads without requiring external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable, network-safe integer value for the enum constant. |
| fromValue(int value) | EmitShape | O(1) | Deserializes an integer into its corresponding EmitShape instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is deserializing a shape identifier from a network stream and using it in game logic.

```java
// Deserialize a shape identifier from a network packet
int shapeId = packet.readVarInt();
EmitShape shape = EmitShape.fromValue(shapeId);

// Use the enum in a switch statement for type-safe logic
switch (shape) {
    case Sphere:
        // Handle sphere-specific particle emission logic
        break;
    case Cube:
        // Handle cube-specific particle emission logic
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Usage for Serialization:** Do not use the built-in `ordinal()` method for serialization. The integer value is explicitly defined and stable, whereas the ordinal can change if the enum declaration order is modified. Relying on `ordinal()` will break network compatibility between different client or server versions.
- **Ignoring ProtocolException:** Do not wrap calls to fromValue in a generic `try-catch (Exception e)`. The ProtocolException is a specific, unchecked exception indicating a data corruption or protocol mismatch. It should be allowed to propagate up to the network layer's central error handler, which will typically terminate the connection.

## Data Pipeline
EmitShape acts as a deserialization and validation step for shape data flowing from the network into the game engine.

> Flow:
> Network Byte Stream -> Protocol Decoder -> **EmitShape.fromValue(intValue)** -> Game Logic (e.g., Particle System)


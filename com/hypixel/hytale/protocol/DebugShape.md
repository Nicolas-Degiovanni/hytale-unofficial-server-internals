---
description: Architectural reference for DebugShape
---

# DebugShape

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum DebugShape {
```

## Architecture & Concepts
The DebugShape enum is a foundational component of the Hytale network protocol, specifically for server-to-client debug visualization. It serves as a type-safe and protocol-stable contract for defining the geometric shapes that can be rendered by the client for debugging purposes.

Its primary architectural role is to decouple the high-level concept of a shape (e.g., Sphere) from its low-level representation on the wire (e.g., the integer 0). By providing explicit integer values, it ensures that the network protocol remains stable even if the order of enum constants is changed in the source code, a common source of bugs when relying on the default enum ordinal.

The `fromValue` factory method acts as a deserialization gatekeeper, immediately validating incoming data and throwing a ProtocolException upon encountering an undefined shape value. This prevents invalid data from propagating further into the rendering engine.

### Lifecycle & Ownership
- **Creation:** Instances are created and managed exclusively by the Java Virtual Machine (JVM) during class loading. As compile-time constants, they are not instantiated dynamically.
- **Scope:** Application-wide static constants. They persist for the entire lifetime of the application.
- **Destruction:** Unloaded by the JVM during application shutdown. There is no manual memory management required.

## Internal State & Concurrency
- **State:** Inherently immutable. Each enum constant is a singleton instance with a final `value` field. Its state cannot be modified after creation.
- **Thread Safety:** Fully thread-safe. As immutable singletons, instances of DebugShape can be safely accessed and shared across any number of threads without synchronization.

## API Surface
The public contract is focused on serialization and deserialization for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable integer value for serialization. |
| fromValue(int value) | static DebugShape | O(1) | Deserializes an integer into a DebugShape instance. Throws ProtocolException if the value is not defined. |

## Integration Patterns

### Standard Usage
DebugShape is intended to be used within network packet codecs to read and write shape identifiers from a data stream.

```java
// Example: Deserializing a debug command from a network buffer
int shapeId = buffer.readInt();
DebugShape shape = DebugShape.fromValue(shapeId);

// ... use the shape to configure a rendering task

// Example: Serializing a debug command to a network buffer
DebugShape shapeToSend = DebugShape.Cube;
buffer.writeInt(shapeToSend.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Reliance on Ordinal:** Do not use the built-in `ordinal()` method for serialization. It is not protocol-stable and will break compatibility if enum constants are reordered. Always use `getValue()`.
- **Ignoring Exceptions:** Never swallow a ProtocolException from `fromValue`. This exception indicates a critical protocol mismatch or data corruption that must be handled, typically by disconnecting the client.
- **Manual Instantiation:** The Java compiler prevents instantiation of enums with the `new` keyword. They must be accessed as static constants (e.g., DebugShape.Sphere).

## Data Pipeline
DebugShape acts as a data model within the network and rendering pipelines. It does not process data itself but rather represents a state or type.

**Inbound (Server -> Client):**
> Flow:
> Network Buffer (int) -> Protocol Codec -> **DebugShape.fromValue()** -> DebugShape Instance -> Debug Rendering System

**Outbound (Client -> Server, if applicable):**
> Flow:
> Game Logic -> DebugShape Instance -> **DebugShape.getValue()** -> Protocol Codec -> Network Buffer (int)


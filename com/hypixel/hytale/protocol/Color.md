---
description: Architectural reference for Color
---

# Color

**Package:** com.hypixel.hytale.protocol
**Type:** Value Object / Transient

## Definition
```java
// Signature
public class Color {
```

## Architecture & Concepts
The Color class is a fundamental data structure within the Hytale network protocol layer. It is not a general-purpose rendering or utility class; its design is exclusively optimized for high-performance network serialization and deserialization.

As a Plain Old Java Object (POJO), its primary function is to represent a simple 24-bit RGB color value. The class is a component of a larger, highly structured protocol system, evidenced by its static constants like FIXED_BLOCK_SIZE and MAX_SIZE. These constants are used by higher-level protocol message parsers to calculate buffer offsets and validate packet integrity without needing to instantiate the object itself.

This class acts as a data contract between the client and server for any feature requiring color information, such as character customization, block tints, or particle effects.

## Lifecycle & Ownership
- **Creation:** An instance of Color is created under two primary circumstances:
    1. **Outbound:** Application logic instantiates a Color object to populate a field in a larger protocol message that is about to be sent over the network.
    2. **Inbound:** The static factory method deserialize is called by the network protocol decoder when parsing an incoming data stream from a Netty ByteBuf.
- **Scope:** Color objects are transient and short-lived. Their lifecycle is typically bound to the lifecycle of the containing protocol message. Once the message is processed and its data is transferred to the core game state, the Color object is no longer referenced.
- **Destruction:** The object is managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
- **State:** The internal state is defined by three public byte fields: red, green, and blue. The state is fully mutable, allowing direct modification after construction. This design prioritizes performance and ease of use within the single-threaded context of the network or game loop over encapsulation.
- **Thread Safety:** This class is **not thread-safe**. The public, mutable fields make it susceptible to race conditions if accessed and modified concurrently by multiple threads. It is designed with the expectation that it will be handled by a single thread at a time, such as a Netty event loop thread or the main game logic thread.

**WARNING:** Do not share instances of Color between threads without explicit external synchronization. Doing so will lead to unpredictable behavior and data corruption.

## API Surface
The public API is minimal, focusing entirely on serialization, validation, and value-based operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static Color | O(1) | Constructs a new Color object by reading 3 bytes from the given buffer at the specified offset. |
| serialize(ByteBuf) | void | O(1) | Writes the red, green, and blue byte values sequentially into the provided buffer. |
| computeSize() | int | O(1) | Returns the fixed size of the serialized object, which is always 3 bytes. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if the buffer has at least 3 readable bytes from the given offset. Does not perform content validation. |
| clone() | Color | O(1) | Creates a shallow copy of the Color object. |

## Integration Patterns

### Standard Usage
The Color class is intended to be used as a field within a larger protocol message object. The parent object's serialization and deserialization logic is responsible for invoking the corresponding methods on the Color instance.

```java
// Example of a parent message deserializing a Color field
public void deserialize(ByteBuf buf, int offset) {
    // ... deserialize other fields ...
    this.entityColor = Color.deserialize(buf, someCalculatedOffset);
    // ...
}

// Example of a parent message serializing a Color field
public void serialize(ByteBuf buf) {
    // ... serialize other fields ...
    this.entityColor.serialize(buf);
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Use in Rendering Engine:** Do not pass this class directly to the rendering engine. The renderer typically expects color data in a floating-point format (e.g., 0.0 to 1.0). This class uses bytes (0 to 255) and is not suited for graphics calculations. Convert it to a renderer-specific format first.
- **Concurrent Modification:** Do not modify a Color object from one thread while it is being read or serialized by another. This will cause data races.
- **Relying on Identity:** Never use reference equality (==) to compare two Color objects. Always use the provided equals method for value-based comparison.

## Data Pipeline
The Color class is a simple node in the network data pipeline.

> **Outbound Flow:**
> Game State Change -> Protocol Message Creation -> **new Color(r, g, b)** -> Message.serialize() -> **Color.serialize(buf)** -> Netty ByteBuf -> Network Socket

> **Inbound Flow:**
> Network Socket -> Netty ByteBuf -> Protocol Message Deserializer -> **Color.deserialize(buf, offset)** -> Game State Update


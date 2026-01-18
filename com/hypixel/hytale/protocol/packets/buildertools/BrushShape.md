---
description: Architectural reference for BrushShape
---

# BrushShape

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Utility / Enum

## Definition
```java
// Signature
public enum BrushShape {
```

## Architecture & Concepts
The BrushShape enum serves as a type-safe, immutable representation of the geometric shapes available in the game's builder tools. Its primary architectural role is to act as a contract within the network protocol layer, ensuring that both the client and server have a consistent and unambiguous understanding of a brush's shape.

By mapping human-readable names like Sphere and Cube to fixed integer values, this enum prevents the use of "magic numbers" in the networking code. It is a critical component for the serialization and deserialization of packets related to world modification, providing a robust and maintainable way to transmit builder tool state over the network. It is fundamentally a Data Transfer Object (DTO) value, not a service or manager.

### Lifecycle & Ownership
- **Creation:** BrushShape constants are instantiated by the Java Virtual Machine (JVM) during class loading. Developers do not and cannot manually instantiate this enum. They must use the predefined static constants, for example, BrushShape.Cube.
- **Scope:** Application-wide. Once the BrushShape class is loaded by the JVM, its constants exist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are garbage collected by the JVM only when the application is shutting down and its classloader is unloaded.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a single, final integer field named *value*. The public static VALUES array is also final. The state of this object cannot be modified after its creation by the JVM.
- **Thread Safety:** This class is inherently thread-safe. Its immutability and the JVM's guarantees for enum initialization ensure that it can be safely accessed and used from any thread without synchronization. The static factory method fromValue is a pure function and is also completely thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value assigned to the enum constant for network serialization. |
| fromValue(int value) | BrushShape | O(1) | **Static Factory.** Deserializes an integer from a network stream into its corresponding BrushShape constant. Throws ProtocolException if the value is out of bounds. |
| VALUES | BrushShape[] | N/A | **Static Field.** A cached, public array of all BrushShape constants. This is an optimization to prevent repeated array allocation from the internal values() method. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the encoding and decoding of network packets. The encoder uses getValue to serialize the shape to an integer, and the decoder uses fromValue to deserialize it back into the type-safe enum.

```java
// Example: Encoding a packet
int shapeId = BrushShape.Sphere.getValue();
packetBuffer.writeVarInt(shapeId);

// Example: Decoding a packet
int receivedId = packetBuffer.readVarInt();
BrushShape shape = BrushShape.fromValue(receivedId);
// ... use the 'shape' object in game logic
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinal Values:** Never rely on the built-in `ordinal()` method for serialization. The explicit `value` field is the serialization contract. The order of enum declarations can change, which would break the protocol if `ordinal()` were used. This class correctly provides `getValue()`.
- **Unsafe Deserialization:** The `fromValue` method will throw a runtime ProtocolException for invalid data. Any code decoding network packets must be prepared to handle this exception to prevent a client or server crash due to malformed or malicious packets.
- **Direct Instantiation:** Enums cannot be instantiated with `new`. Attempting to do so will result in a compile-time error.

## Data Pipeline
BrushShape is a data model that flows through the network pipeline. It does not process data itself but is a fundamental part of the data being processed.

> **Serialization Flow (Client to Server):**
> Builder Tool UI Event -> Game Logic creates Packet with `BrushShape.Cylinder` -> Packet Encoder calls `getValue()` -> Integer `2` written to Network Stream

> **Deserialization Flow (Server from Client):**
> Integer `2` read from Network Stream -> Packet Decoder calls `fromValue(2)` -> Returns `BrushShape.Cylinder` -> Game Logic receives Packet with type-safe enum<ctrl63>


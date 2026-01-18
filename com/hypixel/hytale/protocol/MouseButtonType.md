---
description: Architectural reference for MouseButtonType
---

# MouseButtonType

**Package:** com.hypixel.hytale.protocol
**Type:** Data Type

## Definition
```java
// Signature
public enum MouseButtonType {
```

## Architecture & Concepts
The **MouseButtonType** enum is a foundational data type within the Hytale network protocol. Its primary function is to provide a compile-time, type-safe representation of discrete mouse button inputs. This component is critical for standardizing how player actions are serialized, transmitted over the network, and deserialized by the server.

By replacing ambiguous "magic numbers" (e.g., 0 for left-click) with named constants like **MouseButtonType.Left**, the enum significantly improves the readability, reliability, and maintainability of the entire input-handling and networking stack.

The static factory method, **fromValue**, acts as the primary deserialization entry point. It is responsible for converting a raw integer received from a network stream into a valid enum instance. This method includes strict bounds checking, throwing a **ProtocolException** if an undefined integer value is received. This behavior is a core part of the protocol's defensive design, ensuring that corrupted or malicious packets are rejected early in the processing pipeline.

## Lifecycle & Ownership
- **Creation:** Enum constants are special objects instantiated by the Java Virtual Machine (JVM) during class loading. They are created once and exist as static, singleton instances. Developers do not and cannot manually instantiate this type.
- **Scope:** Application-wide. The enum constants (**Left**, **Middle**, etc.) are available for the entire lifetime of the application.
- **Destruction:** The instances are managed entirely by the JVM and are reclaimed only during application shutdown. There is no manual memory management or destruction required.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final primitive integer value that is assigned at creation and can never be changed. The static **VALUES** array is also final, providing a stable, cached list for deserialization lookups.
- **Thread Safety:** This type is inherently thread-safe. Its immutability and the JVM's guarantees for enum initialization ensure that it can be safely accessed and shared across any number of threads without external synchronization or locking mechanisms.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the primitive integer value for network serialization. |
| fromValue(int value) | static MouseButtonType | O(1) | Deserializes an integer into an enum constant. Throws **ProtocolException** if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case involves serializing an enum to an integer for network transmission and deserializing it back on the receiving end.

**WARNING:** The **fromValue** method can throw a checked exception. Calling code in the network layer *must* handle **ProtocolException** to prevent unhandled packet processing failures.

```java
// Server-side: Deserializing from a network packet
int buttonId = packet.readVarInt();
MouseButtonType receivedButton;
try {
    receivedButton = MouseButtonType.fromValue(buttonId);
} catch (ProtocolException e) {
    // Handle invalid packet data, e.g., disconnect the client
    log.error("Invalid MouseButtonType received: " + buttonId);
    return;
}

// Client-side: Serializing for network transmission
int buttonIdToSend = MouseButtonType.Left.getValue();
packet.writeVarInt(buttonIdToSend);
```

### Anti-Patterns (Do NOT do this)
- **Integer Comparison:** Avoid comparing enum instances by their underlying integer value. This defeats the purpose of type safety and makes the code brittle. Use direct object comparison, which is guaranteed to work for enums.

```java
// BAD: Prone to error and hard to read
if (button.getValue() == 0) {
    // ... do left click action
}

// GOOD: Type-safe, readable, and performant
if (button == MouseButtonType.Left) {
    // ... do left click action
}
```

- **Ignoring Exceptions:** Never call **fromValue** without a try-catch block in production code. An invalid value will halt the processing of a network packet if the exception is not handled.

## Data Pipeline
**MouseButtonType** is not a processing component but rather a data model that represents state within the pipeline. It serves as the canonical representation for mouse inputs after they are captured and before they are serialized, and vice-versa.

> **Client-Side Flow:**
> Raw OS Input -> InputManager -> **MouseButtonType** constant -> Network Packet Encoder -> `getValue()` -> Raw Integer on Wire

> **Server-Side Flow:**
> Raw Integer on Wire -> Network Packet Decoder -> `fromValue()` -> **MouseButtonType** constant -> Game Logic System


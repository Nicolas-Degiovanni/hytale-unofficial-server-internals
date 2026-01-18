---
description: Architectural reference for EasingType
---

# EasingType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum EasingType {
```

## Architecture & Concepts
The EasingType enum defines a standardized set of mathematical easing functions used for interpolation and animation throughout the game engine. Its placement within the `com.hypixel.hytale.protocol` package is a critical architectural decision, indicating that these types are part of the network contract between the client and server. This ensures that animations and other time-based transitions can be synchronized deterministically across the network.

Each enum constant is mapped to a final integer value. This integer is the canonical representation used for serialization, providing a compact and efficient on-the-wire format. The primary function of this enum is to decouple the abstract concept of an animation curve (e.g., *CubicInOut*) from its concrete mathematical implementation, while providing a type-safe mechanism for network communication.

## Lifecycle & Ownership
- **Creation:** Enum instances are constants created by the Java Virtual Machine during class loading. They are not instantiated dynamically at runtime. The static `VALUES` array is initialized once at the same time.
- **Scope:** As static constants, all EasingType instances exist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** Instances are destroyed only when the application's class loader is garbage collected, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** EasingType is immutable. The integer `value` associated with each constant is a `final` field assigned at creation. The state of an EasingType cannot be changed after initialization.
- **Thread Safety:** This enum is intrinsically thread-safe due to its immutability. It can be safely accessed and read from any thread without requiring external synchronization or locks. The static `fromValue` method is also thread-safe as it operates on the final `VALUES` array.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value used for network serialization. |
| fromValue(int value) | EasingType | O(1) | **Static.** Deserializes an integer into its corresponding EasingType. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is used to serialize and deserialize animation curve types for network transmission. The `getValue` and `fromValue` methods form a symmetric pair for this purpose.

```java
// Example: Serializing an animation instruction for the network
int serializedType = EasingType.ElasticOut.getValue();
// network.send(serializedType);

// Example: Deserializing an animation instruction from the network
// int receivedType = network.readInt();
int receivedType = 23; // Simulating received data
EasingType easing = EasingType.fromValue(receivedType);
// animationSystem.apply(easing);
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Do not use the built-in `ordinal()` method for serialization. The integer value assigned in the constructor is the canonical network value. The order of declaration, and thus the ordinal, could change between versions, breaking network compatibility. Always use `getValue()`.
- **Invalid Value Handling:** The `fromValue` method throws a runtime ProtocolException on invalid input. Network decoding logic must be prepared to handle this exception, typically by disconnecting the client to prevent state corruption.

## Data Pipeline
EasingType acts as a data contract within the network pipeline, translating between a high-level game concept and its low-level integer representation.

> **Serialization Flow:**
> Animation System -> **EasingType.BackInOut** -> `getValue()` -> `int 27` -> Protocol Buffer -> Network Packet

> **Deserialization Flow:**
> Network Packet -> Protocol Buffer -> `int 27` -> `fromValue(27)` -> **EasingType.BackInOut** -> Animation System


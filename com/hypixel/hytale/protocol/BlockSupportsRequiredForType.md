---
description: Architectural reference for BlockSupportsRequiredForType
---

# BlockSupportsRequiredForType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility Enum

## Definition
```java
// Signature
public enum BlockSupportsRequiredForType {
```

## Architecture & Concepts
BlockSupportsRequiredForType is a type-safe enumeration that defines a logical condition for block support mechanics within the Hytale protocol. It is not a dynamic system but rather a static data contract, ensuring that both the client and server interpret block placement rules identically.

This enum is a critical component of the block definition data model. When the server transmits block data to a client, this value dictates how the client's physics and rendering engine should interpret the block's stability requirements. For example, it determines whether a torch needs to be attached to *any* valid surface or if a more complex block requires support from *all* specified adjacent faces.

The inclusion of a custom `ProtocolException` in the `fromValue` factory method underscores its role as a validation gateway. It actively prevents data corruption by rejecting any integer value that does not map to a defined enum constant, thereby enforcing protocol integrity at the earliest possible stage of data deserialization.

### Lifecycle & Ownership
- **Creation:** Enum instances are constants, instantiated by the Java Virtual Machine during class loading. The two instances, Any and All, are created once and only once.
- **Scope:** Application-wide. These instances persist for the entire lifetime of the JVM process. They are effectively global, immutable singletons.
- **Destruction:** The enum and its instances are eligible for garbage collection only when the application's ClassLoader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a private final integer value that is assigned at creation and can never be changed. The static VALUES array is also final and is populated once during class initialization.
- **Thread Safety:** This class is inherently thread-safe. Its immutable nature guarantees that it can be safely accessed and used by multiple threads simultaneously without any need for external synchronization or locks. The `fromValue` and `getValue` methods are pure and free of side effects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value associated with the enum constant, intended for network serialization. |
| fromValue(int value) | static BlockSupportsRequiredForType | O(1) | Deserializes an integer from the protocol into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of block data. Logic should always operate on the enum type, converting to and from integers only at the protocol boundary.

```java
// Deserializing from a network packet
int supportTypeValue = networkBuffer.readVarInt();
BlockSupportsRequiredForType supportType = BlockSupportsRequiredForType.fromValue(supportTypeValue);

// Using the value in game logic
if (supportType == BlockSupportsRequiredForType.All) {
    // ... handle logic for blocks requiring all supports
}

// Serializing for a network packet
networkBuffer.writeVarInt(supportType.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Never use the raw integer values 0 or 1 directly in game logic. This defeats the purpose of a type-safe enum, introduces ambiguity, and makes the code brittle. Always compare against the enum constants, for example, `if (type == BlockSupportsRequiredForType.Any)`.
- **Ignoring Exceptions:** The `fromValue` method can throw a `ProtocolException`. Failure to catch this exception can lead to a client or server crash when encountering malformed or malicious network data. Always wrap deserialization logic in a try-catch block.

## Data Pipeline
BlockSupportsRequiredForType acts as a deserialized data model within the protocol pipeline. It translates a low-level integer from a raw data stream into a high-level, type-safe object for consumption by the rest of the game engine.

> Flow:
> Network Byte Stream -> Protocol Deserializer -> `fromValue(intValue)` -> **BlockSupportsRequiredForType** instance -> Block Definition System -> Game Logic (e.g., Block Placement Validator)


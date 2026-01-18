---
description: Architectural reference for OverlapBehavior
---

# OverlapBehavior

**Package:** com.hypixel.hytale.protocol
**Type:** Utility / Value Object

## Definition
```java
// Signature
public enum OverlapBehavior {
```

## Architecture & Concepts
The OverlapBehavior enum is a core component of the Hytale network protocol layer. It provides a type-safe representation for a set of predefined strategies that dictate how the server or client should resolve operations on overlapping data regions or time intervals.

Its primary architectural function is to decouple game logic from the low-level integer values used in network packets. By translating a wire-format integer (e.g., 0, 1, 2) into a self-documenting, immutable constant (e.g., Extend, Overwrite, Ignore), it eliminates "magic numbers" from the codebase. This improves readability, reduces the risk of protocol-related bugs, and centralizes the logic for serialization and deserialization of this specific protocol field.

The three defined behaviors are:
*   **Extend:** The new operation should append to or extend the existing data.
*   **Overwrite:** The new operation should completely replace the existing data.
*   **Ignore:** The new operation should be discarded if it conflicts with existing data.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (Extend, Overwrite, Ignore) are constructed by the Java Virtual Machine during class loading. They are compile-time constants, not runtime objects in the traditional sense.
- **Scope:** As static constants, they exist for the entire lifetime of the application. They are effectively global singletons managed by the JVM.
- **Destruction:** The instances are garbage collected only when the application's ClassLoader is unloaded, which typically occurs at JVM shutdown. No manual memory management is ever required.

## Internal State & Concurrency
- **State:** OverlapBehavior is **deeply immutable**. Each enum constant holds a private final integer value that is assigned at creation and can never be changed. The static VALUES array is also final and its contents are immutable.
- **Thread Safety:** This class is **unconditionally thread-safe**. As immutable singletons, its instances can be safely passed between and accessed by any number of threads without requiring locks or any other synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant. This value is used for serialization into the network protocol. |
| fromValue(int value) | OverlapBehavior | O(1) | A static factory method for deserialization. Translates a raw integer from a network packet into the corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during network packet serialization and deserialization. Logic should use the enum constants directly, converting to and from integers only at the protocol boundary.

```java
// Deserializing a packet from a network buffer
int behaviorValue = buffer.readVarInt();
OverlapBehavior behavior = OverlapBehavior.fromValue(behaviorValue);

// Using the type-safe enum in game logic
switch (behavior) {
    case Overwrite:
        // ... handle overwrite logic
        break;
    case Extend:
        // ... handle extend logic
        break;
}

// Serializing a packet to a network buffer
Packet packet = new Packet();
packet.setOverlap(OverlapBehavior.Overwrite);
buffer.writeVarInt(packet.getOverlap().getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The protocol contract is bound to the explicit integer `value` field. Relying on declaration order via `ordinal()` is fragile and will break the protocol if the enum constants are ever reordered.
- **Integer Comparison:** Avoid comparing behaviors using their integer values in game logic. This defeats the purpose of a type-safe enum. Always compare the enum instances directly using `==`.

    ```java
    // BAD: Re-introduces magic numbers
    if (behavior.getValue() == 1) { ... }

    // GOOD: Type-safe and readable
    if (behavior == OverlapBehavior.Overwrite) { ... }
    ```

## Data Pipeline
OverlapBehavior acts as a translation point between the raw network stream and the type-safe domain of the game engine.

> Flow:
> Network Packet (int) -> Protocol Deserializer -> **OverlapBehavior.fromValue(int)** -> Game Logic (uses enum instance) -> Protocol Serializer -> **OverlapBehavior.getValue()** -> Network Packet (int)


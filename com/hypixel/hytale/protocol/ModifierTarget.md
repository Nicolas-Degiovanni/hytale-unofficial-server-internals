---
description: Architectural reference for ModifierTarget
---

# ModifierTarget

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Constant

## Definition
```java
// Signature
public enum ModifierTarget {
```

## Architecture & Concepts
ModifierTarget is a fundamental type within the Hytale network protocol layer. Its primary function is to provide a compile-time safe, self-documenting representation for a protocol field that can only hold a small, fixed set of values.

This enum acts as a translation layer between the raw integer values transmitted over the network (for efficiency) and the high-level, readable constants used within the game logic. By mapping integers like 0 and 1 to meaningful names like Min and Max, it eliminates the use of "magic numbers", thereby improving code clarity and reducing the risk of bugs.

The static factory method, fromValue, serves as a critical deserialization and validation gateway. It ensures that any integer read from a network stream conforms to the expected protocol definition before it is propagated into the wider system.

## Lifecycle & Ownership
- **Creation:** The enum constants, Min and Max, are instantiated automatically by the Java Virtual Machine (JVM) when the ModifierTarget class is first loaded. This process is managed entirely by the class loader and occurs only once.
- **Scope:** Instances are static and persist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** Resources are reclaimed by the JVM during application shutdown. No manual cleanup is ever required.

## Internal State & Concurrency
- **State:** Inherently immutable. The internal integer value associated with each constant is declared final and is assigned at creation time. The state of a ModifierTarget instance can never change after it is constructed.
- **Thread Safety:** This class is unconditionally thread-safe. As an immutable type, its instances can be freely shared and read by multiple threads concurrently without any need for external synchronization or locking mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation of the constant, primarily for serialization into a network packet. |
| fromValue(int value) | ModifierTarget | O(1) | A static factory that deserializes an integer into a ModifierTarget instance. Throws ProtocolException if the provided value is out of the defined range. |

## Integration Patterns

### Standard Usage
This enum is intended to be used during the network packet decoding process to convert a raw integer into a safe type, which is then passed to game logic systems.

```java
// Example within a packet deserializer
int rawValue = networkBuffer.readVarInt();
ModifierTarget target = ModifierTarget.fromValue(rawValue);

// Pass the 'target' instance to game systems, which can use it safely
applyAttributeModifier(entity, attribute, target);
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integers:** Avoid passing the integer representation of this enum into core game logic. This defeats the purpose of type safety and reintroduces "magic numbers".

    ```java
    // BAD: Logic is unclear and brittle
    if (modifierTargetValue == 0) {
        // ...
    }
    ```

- **Ignoring Deserialization Errors:** The ProtocolException thrown by fromValue indicates a critical error, such as data corruption or a client/server version mismatch. It must not be ignored.

    ```java
    // DANGEROUS: Hides a potentially severe network issue
    try {
        ModifierTarget target = ModifierTarget.fromValue(value);
    } catch (ProtocolException e) {
        // Silently ignoring this can lead to undefined behavior
    }
    ```

## Data Pipeline
ModifierTarget acts as a validation and typing step during the data ingress pipeline, converting raw network data into a structured, reliable format for the game engine.

> Flow:
> Network Byte Stream -> Protocol Decoder -> Raw int -> **ModifierTarget.fromValue()** -> Validated ModifierTarget Instance -> Game Logic System


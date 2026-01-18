---
description: Architectural reference for Rangef
---

# Rangef

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Rangef {
```

## Architecture & Concepts
The **Rangef** class is a fundamental data structure within the Hytale network protocol layer. It represents a simple, inclusive floating-point range defined by a minimum and a maximum value. Its primary architectural role is to serve as a high-performance Data Transfer Object (DTO) for network serialization and deserialization operations.

The design is heavily optimized for performance and memory predictability. It is a fixed-size, 8-byte structure, corresponding directly to two 4-byte floating-point numbers. This fixed-size nature allows the network layer to perform highly efficient, zero-allocation reads and validation.

The static methods, such as **deserialize** and **validateStructure**, are part of a standardized protocol generation pattern. This pattern enables the higher-level network framework to process protocol objects generically without prior instantiation, significantly reducing overhead during packet decoding.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand. The primary creation path is via the static **deserialize** method, which is invoked by the network protocol decoder when a **Rangef** is read from a **ByteBuf**. Game logic can also instantiate it directly using its constructors (e.g., `new Rangef(10.0f, 20.0f)`) when preparing data for serialization.

- **Scope:** Short-lived and transient. The lifetime of a **Rangef** object is typically bound to the scope of a single network packet's processing or a specific game logic calculation. It is not managed by any service container or registry.

- **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as they are no longer referenced, which is typically after the parent network packet or game event has been fully processed.

## Internal State & Concurrency
- **State:** The state is fully defined by its two public fields: **min** and **max**. The class is **mutable**, as these fields can be directly modified after an object has been created. This design choice prioritizes performance over immutability, avoiding the overhead of creating new objects for every change.

- **Thread Safety:** **Rangef** is **not thread-safe**. Its mutable public fields make it inherently unsafe for concurrent modification or for read/write access across different threads without external synchronization.

    **WARNING:** Instances of **Rangef** should be confined to a single thread, such as a Netty I/O thread or the main game update thread. Sharing an instance across threads without explicit locking will lead to race conditions and unpredictable behavior.

## API Surface
The public API is minimal, focusing on serialization, state, and value-object semantics.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static Rangef | O(1) | Constructs a new **Rangef** by reading 8 bytes from the given buffer at the specified offset. |
| serialize(ByteBuf) | void | O(1) | Writes the **min** and **max** fields as little-endian floats into the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check to ensure the buffer contains at least 8 readable bytes at the offset. |
| computeSize() | int | O(1) | Returns the constant serialized size of the object, which is always 8. |
| clone() | Rangef | O(1) | Creates a new **Rangef** instance with the same **min** and **max** values. |

## Integration Patterns

### Standard Usage
A developer typically creates a **Rangef** to define a data range and passes it to a system that will handle its serialization, such as a packet constructor.

```java
// Example: Defining a weapon's damage range for a game event.
// This object would then be added to a larger packet structure
// which is subsequently serialized and sent over the network.

float minDamage = 50.0f;
float maxDamage = 75.5f;

Rangef weaponDamageRange = new Rangef(minDamage, maxDamage);

// The network system would later call serialize() on this object.
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Do not store a **Rangef** instance in a shared cache or pass it between threads without proper synchronization. Its public mutable fields make this pattern extremely dangerous. Create a new instance or clone it instead.
- **Ignoring Validation:** Never call **deserialize** on a buffer received from an untrusted source without first calling **validateStructure**. Failure to do so can result in buffer read exceptions and potential server instability.
- **Assuming Immutability:** Do not pass a **Rangef** to a method and assume its values will not change. Any system with a reference to the object can modify its public **min** and **max** fields.

## Data Pipeline
As a DTO, **Rangef** is the data itself, flowing through the network pipeline. It does not process data but is rather the payload being processed.

> **Deserialization Flow:**
> Raw Network ByteBuf -> Protocol Decoder -> **Rangef.deserialize()** -> **Rangef instance** -> Game Logic System

> **Serialization Flow:**
> Game Logic System -> **new Rangef()** -> Protocol Encoder -> **Rangef.serialize()** -> Raw Network ByteBuf


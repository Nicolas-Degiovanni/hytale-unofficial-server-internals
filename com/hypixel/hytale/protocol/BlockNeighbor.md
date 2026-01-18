---
description: Architectural reference for BlockNeighbor
---

# BlockNeighbor

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum BlockNeighbor {
```

## Architecture & Concepts
BlockNeighbor is a foundational enumeration that provides a type-safe, compile-time constant representation for the 26 possible neighbor directions of a block within a three-dimensional grid. It serves as a core component of the world and protocol systems, replacing ambiguous "magic numbers" with explicit, human-readable identifiers like North, Up, and DownSouthWest.

This enum is critical for network serialization and deserialization. Each constant is mapped to a unique, stable integer value. This allows for a highly compact and efficient representation of directional data when sent over the network or saved to disk. The static `fromValue` method acts as the deserialization factory, converting the integer back into its corresponding enum object, while `getValue` performs serialization.

Architecturally, BlockNeighbor is a fundamental data type used extensively in algorithms for:
- **World Generation:** Determining block placement based on adjacent blocks.
- **Block Updates:** Propagating changes (e.g., fluid flow, redstone signals) to neighbors.
- **Rendering:** Culling unseen faces and calculating ambient occlusion.
- **Physics:** Collision detection and entity movement.

The static `VALUES` array is a performance optimization. It provides a pre-cached array of all enum constants, allowing the `fromValue` method to perform a direct array lookup instead of repeatedly calling the `values()` method, which would allocate a new array on each invocation.

## Lifecycle & Ownership
- **Creation:** All 26 BlockNeighbor constants are instantiated by the Java Virtual Machine during class loading. This process is automatic and occurs before any game code directly references the enum.
- **Scope:** Application-wide. The enum constants are static and persist for the entire lifetime of the application.
- **Destruction:** The constants are garbage collected only when the JVM shuts down. There is no manual memory management associated with this type.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state of each enum constant, specifically its integer `value`, is final and cannot be modified after its creation by the JVM.
- **Thread Safety:** This class is unconditionally **thread-safe**. As immutable objects, enum constants can be safely shared and read by any number of threads without requiring synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable integer value associated with the neighbor direction, used for serialization. |
| fromValue(int value) | BlockNeighbor | O(1) | **[Factory]** Deserializes an integer into a BlockNeighbor constant. Throws ProtocolException if the value is out of the valid range [0-25]. |

## Integration Patterns

### Standard Usage
The primary interaction pattern involves converting between the enum and its integer representation during data serialization and deserialization, typically within the network protocol layer.

```java
// Deserialization from a network packet or data file
int directionId = readIntFromBuffer(); // e.g., reads the value 3
BlockNeighbor direction = BlockNeighbor.fromValue(directionId); // Returns BlockNeighbor.East

// Serialization to a network packet
BlockNeighbor neighborToUpdate = BlockNeighbor.UpNorth;
writeIntToBuffer(neighborToUpdate.getValue()); // Writes the value 6
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Do not use the built-in `ordinal()` method to get the integer value. The `value` field is the canonical, stable identifier for serialization. The order of constants in the source file could change, which would break all serialized data if `ordinal()` were used. Always use `getValue()`.
- **Ignoring Exceptions:** The `fromValue` method can throw a ProtocolException. This is a critical security and stability boundary. Failure to handle this exception can lead to crashes or corrupted game state if invalid data is received from a client or a file.

## Data Pipeline
BlockNeighbor acts as a translation layer between the high-level game logic and the low-level serialized data format.

> **Serialization Flow:**
> Game Logic (e.g., BlockUpdateEvent) -> **BlockNeighbor.getValue()** -> Integer [0-25] -> Network Buffer

> **Deserialization Flow:**
> Network Buffer -> Integer [0-25] -> **BlockNeighbor.fromValue()** -> Game Logic (e.g., World.setBlock)


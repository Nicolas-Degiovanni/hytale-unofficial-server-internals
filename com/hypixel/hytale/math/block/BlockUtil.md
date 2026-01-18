---
description: Architectural reference for BlockUtil
---

# BlockUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockUtil {
```

## Architecture & Concepts
BlockUtil is a low-level, foundational utility class responsible for the serialization and deserialization of three-dimensional block coordinates. Its primary function is to convert a set of integer coordinates (x, y, z) into a single, compact 64-bit long, and vice-versa.

This component is critical for engine performance and memory efficiency. Storing a block position as a `long` (8 bytes) is significantly more efficient than using a Vector3i object, which carries object overhead in addition to its three 4-byte integer fields. This optimization is essential for systems that must manage immense collections of block positions, such as world generation, chunk data storage, network synchronization packets, and large-scale physics calculations.

BlockUtil establishes the canonical packed representation for block coordinates throughout the entire engine. Any system that needs to store or transmit a block position in a compact form must defer to this utility to ensure system-wide compatibility.

The specific bitwise layout is as follows:
- **X Coordinate:** 27 bits (26 for value, 1 for sign)
- **Z Coordinate:** 27 bits (26 for value, 1 for sign)
- **Y Coordinate:** 10 bits (9 for value, 1 for sign)

This layout imposes strict boundaries on world coordinates, which are enforced by the `pack` methods.

## Lifecycle & Ownership
As a static utility class, BlockUtil is never instantiated and therefore has no lifecycle in the traditional object-oriented sense.

- **Creation:** Not applicable. The class is loaded into the JVM by the ClassLoader at runtime, typically upon first access.
- **Scope:** Application-wide. Its static methods are globally accessible once the class is loaded.
- **Destruction:** Not applicable. The class is unloaded from the JVM when the application terminates.

## Internal State & Concurrency
- **State:** BlockUtil is entirely **stateless**. It contains no mutable fields, either static or instance. All its fields are compile-time constants that define the bit masks and coordinate limits. Each method call operates exclusively on its input arguments.
- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, its methods can be invoked concurrently from any number of threads without risk of data corruption or race conditions. No synchronization mechanisms, such as locks, are necessary.

## API Surface
The public contract consists of static methods for packing and unpacking coordinates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| pack(int x, int y, int z) | long | O(1) | Packs three integer coordinates into a single long. Throws IllegalArgumentException if any coordinate is outside the valid range. |
| pack(Vector3i val) | long | O(1) | Convenience overload that delegates to `pack(int, int, int)`. |
| unpack(long packed) | Vector3i | O(1) | Unpacks a long into a new Vector3i object. |
| unpackX(long packed) | int | O(1) | Extracts only the X coordinate from a packed long. |
| unpackY(long packed) | int | O(1) | Extracts only the Y coordinate from a packed long. |
| unpackZ(long packed) | int | O(1) | Extracts only the Z coordinate from a packed long. |

## Integration Patterns

### Standard Usage
The primary use case is to convert between object-oriented coordinate representations and their primitive, packed form for efficient storage or transmission.

```java
// A block position in the game world
Vector3i blockPosition = new Vector3i(1024, 64, -2048);

// Pack the position into a long for storage or networking
long packedPosition = BlockUtil.pack(blockPosition);

// ... later, or on another system ...

// Unpack the long back into a usable Vector3i object
Vector3i restoredPosition = BlockUtil.unpack(packedPosition);
```

### Anti-Patterns (Do NOT do this)
- **Re-implementing Packing Logic:** Do not write custom bit-shifting logic to pack or unpack coordinates. BlockUtil provides the single, canonical implementation for the entire engine. Deviating from this will lead to data corruption and incompatibility between systems.
- **Ignoring Coordinate Limits:** The `pack` method will throw an exception for out-of-range coordinates. Do not rely on a try-catch block for standard control flow. Instead, game logic should validate or clamp coordinates *before* attempting to pack them.
- **Inefficient Storage:** Do not store collections of block positions, such as in a HashSet or HashMap, using Vector3i as the key. The packed `long` primitive is significantly more memory-efficient and generates a faster, more uniform hash code.

## Data Pipeline
BlockUtil acts as a pure transformation function within a larger data flow. It does not initiate or terminate a pipeline but is a critical step for serialization and deserialization.

> **Serialization Flow:**
> Game Logic (Vector3i) -> **BlockUtil.pack** -> Packed `long` -> Network Buffer / Disk Storage

> **Deserialization Flow:**
> Network Buffer / Disk Storage -> Packed `long` -> **BlockUtil.unpack** -> Game Logic (Vector3i)


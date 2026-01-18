---
description: Architectural reference for BitConverter
---

# BitConverter

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Utility

## Definition
```java
// Signature
public class BitConverter {
```

## Architecture & Concepts
The BitConverter class is a low-level, stateless utility providing a foundational bridge between primitive numeric types and their binary representation. It serves as a core component for any system requiring direct, high-performance bit-level manipulation, such as network serialization, file I/O, or custom data compression schemes.

This class operates as a pure data transformer. It does not integrate with the game loop, entity component system, or any high-level engine services. Its purpose is to offer a clear, unambiguous API for converting numbers into a boolean array format and back, abstracting away the complexities of bitwise shifting and masking operations. The use of a boolean array as the intermediate format is a design choice favoring readability and ease of manipulation over raw byte buffers in specific contexts.

## Lifecycle & Ownership
As a static utility class, BitConverter does not follow a traditional object lifecycle.

- **Creation:** Not applicable. BitConverter is a static utility class and is never instantiated. Its methods are invoked directly on the class itself.
- **Scope:** Class-level scope. The methods are available globally as soon as the BitConverter class is loaded by the Java Virtual Machine.
- **Destruction:** The class is unloaded by the ClassLoader according to standard JVM garbage collection rules for static resources, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Stateless. The class contains no member fields, and all methods are pure functions. Their output depends exclusively on their input arguments, with no side effects.
- **Thread Safety:** Fully thread-safe. Due to its stateless and pure-functional design, all methods can be invoked concurrently from any thread without risk of race conditions or data corruption. No external synchronization or locking is required when using this class.

## API Surface
The public API consists of paired methods for converting between primitive numbers and boolean arrays.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toBitArray(long number) | boolean[] | O(1) | Converts a 64-bit long into a new, fixed-size boolean array of 64 elements. |
| toBitArray(int number) | boolean[] | O(1) | Converts a 32-bit integer into a new, fixed-size boolean array of 32 elements. |
| toBitArray(byte number) | boolean[] | O(1) | Converts an 8-bit byte into a new, fixed-size boolean array of 8 elements. |
| toLong(boolean[] bits) | long | O(1) | Converts a boolean array into a 64-bit long. Throws IllegalArgumentException if the array length is not exactly 64. |
| toInt(boolean[] bits) | int | O(1) | Converts a boolean array into a 32-bit integer. Throws IllegalArgumentException if the array length is not exactly 32. |
| toByte(boolean[] bits) | int | O(1) | Converts a boolean array into an 8-bit byte, returned as an int. Throws IllegalArgumentException if the array length is not exactly 8. |

## Integration Patterns

### Standard Usage
The class should be used for explicit, controlled conversions where a binary representation is needed for processing or transport.

```java
// Example: Serializing a configuration flag set stored in an integer
int configFlags = 21; // Binary 00010101

// Convert to a bit array for inspection or serialization
boolean[] bits = BitConverter.toBitArray(configFlags);

// ... network transmission or file write ...

// Reconstruct the integer from the bit array
int reconstructedFlags = BitConverter.toInt(bits);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance via `new BitConverter()`. All methods are static and must be called directly on the class, for example, `BitConverter.toInt(myArray)`.
- **Incorrect Array Sizing:** Passing a boolean array of the wrong size to a conversion method is a critical error. For example, providing a 32-element array to toLong will deterministically throw an IllegalArgumentException. Always ensure the array size matches the target data type's precision.
- **Performance-Critical Loops:** While the methods are O(1), creating new boolean arrays in a tight, performance-critical loop can exert pressure on the garbage collector. For extreme performance needs, consider direct bitwise operations on primitive types instead.

## Data Pipeline
BitConverter acts as a bidirectional transformation step within a larger data pipeline. It does not manage the flow of data itself but provides the necessary conversion logic.

> **Serialization Flow:**
> Primitive Number (long, int, byte) -> **BitConverter.toBitArray()** -> `boolean[]` -> Serializer

> **Deserialization Flow:**
> Deserializer -> `boolean[]` -> **BitConverter.toType()** -> Primitive Number (long, int, byte)


---
description: Architectural reference for BitUtil
---

# BitUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class BitUtil {
```

## Architecture & Concepts
BitUtil is a low-level, stateless utility class designed for high-performance, direct memory manipulation of byte arrays. Its sole function is to provide an abstraction for reading and writing 4-bit values, known as nibbles, within a contiguous block of memory represented by a byte array.

In systems where memory footprint and network bandwidth are critical, such as chunk data serialization or network packet construction, data is often packed tightly. This class serves as a foundational component for such systems, abstracting the complex and error-prone bitwise operations required to access data at a sub-byte level. It operates on the principle that a single 8-bit byte can store two distinct 4-bit nibbles, effectively doubling the density of certain types of data storage.

This utility is not part of the core game loop but is a critical dependency for systems responsible for serialization, persistence, and networking.

### Lifecycle & Ownership
- **Creation:** As a static utility class, BitUtil is never instantiated. Its bytecode is loaded into the JVM by the ClassLoader when it is first referenced by another class.
- **Scope:** The class and its static methods are available for the entire application lifetime once loaded.
- **Destruction:** The class is unloaded from the JVM when the application terminates. No explicit cleanup or resource management is required.

## Internal State & Concurrency
- **State:** BitUtil is completely stateless. It contains no member fields and all its methods are pure functions whose output depends solely on their input arguments. The operations are performed directly on the byte array provided by the caller.

- **Thread Safety:** The methods within BitUtil are inherently thread-safe. However, the byte array passed as the `data` parameter is **not** protected. Callers are responsible for ensuring proper synchronization if multiple threads are reading from or writing to the same byte array concurrently. Failure to provide external locking will result in severe data corruption and race conditions.

## API Surface
The public API provides atomic operations for getting, setting, and swapping nibble values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setNibble(data, idx, b) | void | O(1) | Sets a 4-bit value at the specified nibble index. The input value `b` is masked to 4 bits. |
| getNibble(data, idx) | byte | O(1) | Retrieves the 4-bit value from the specified nibble index. |
| getAndSetNibble(data, idx, b) | byte | O(1) | Atomically retrieves the original 4-bit value and replaces it with a new one. |

## Integration Patterns

### Standard Usage
BitUtil should be used by any system that requires dense data packing. A common use case is storing two distinct values ranging from 0-15 (e.g., light levels) within a single byte.

```java
// Example: Storing sky light and block light levels in a single byte
byte[] chunkLightData = new byte[1];

// Set the sky light level (nibble at index 0) to 15 (maximum)
BitUtil.setNibble(chunkLightData, 0, (byte) 15);

// Set the block light level (nibble at index 1) to 7
BitUtil.setNibble(chunkLightData, 1, (byte) 7);

// Later, retrieve the sky light level
byte skyLight = BitUtil.getNibble(chunkLightData, 0); // Returns 15
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Unsynchronized Access:** The most critical anti-pattern is modifying a shared byte array from multiple threads without external synchronization. This will lead to unpredictable data corruption.

    ```java
    // BAD: Race condition waiting to happen
    byte[] sharedData = new byte[1];
    new Thread(() -> BitUtil.setNibble(sharedData, 0, (byte) 5)).start();
    new Thread(() -> BitUtil.setNibble(sharedData, 0, (byte) 10)).start();
    ```

- **Index Confusion:** Do not mistake the `idx` parameter for a byte index. It is a **nibble index**. Nibble indices 0 and 1 both map to the first byte of the array, indices 2 and 3 map to the second, and so on. Using a byte index will lead to incorrect data access.

## Data Pipeline
BitUtil is not a standalone stage in a data pipeline; rather, it is a low-level tool used by serialization and deserialization stages to perform data transformation.

> **Serialization Flow:**
> High-Level Game Object (e.g., Chunk) -> Serializer -> **BitUtil** (packs data like light levels into a compact byte array) -> Network Buffer / Disk File

> **Deserialization Flow:**
> Network Buffer / Disk File -> Deserializer -> **BitUtil** (unpacks nibbles from the byte array) -> High-Level Game Object


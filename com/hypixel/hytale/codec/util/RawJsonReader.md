---
description: Architectural reference for RawJsonReader
---

# RawJsonReader

**Package:** com.hypixel.hytale.codec.util
**Type:** Transient

## Definition
```java
// Signature
public class RawJsonReader implements AutoCloseable {
```

## Architecture & Concepts
The RawJsonReader is a high-performance, low-level, forward-only stream parser for the JSON data format. It serves as a foundational component of the engine's data serialization and asset loading pipeline, sitting directly between raw character streams and higher-level object mappers (Codecs).

Its primary architectural goal is to minimize memory allocations and CPU overhead during the deserialization of large or numerous JSON files. This is achieved through several key strategies:

1.  **Direct Buffer Manipulation:** It operates directly on a primitive `char[]` buffer, avoiding the creation of intermediate String objects for keys and values unless explicitly requested via a method like `readString`.
2.  **Stream-Based Processing:** It can wrap any standard Java `Reader`, allowing it to process files of arbitrary size without loading the entire content into memory. The internal buffer is refilled from the stream on demand.
3.  **Performance-Intensive Optimizations:** The class employs advanced techniques for performance-critical paths. This includes the use of `sun.misc.Unsafe` for bulk memory operations (`readStringPartAsLongUnsafe`) and a specialized, high-speed double parser (`JavaDoubleParser`) instead of standard library equivalents.
4.  **Thread-Local Buffer Pooling:** The static `READ_BUFFER` field utilizes a `ThreadLocal` to maintain a large, reusable character buffer for each worker thread. This drastically reduces garbage collection pressure in multithreaded contexts, such as parallel asset loading, by preventing the constant allocation and deallocation of large arrays.

In essence, RawJsonReader is not a general-purpose JSON library but a specialized tool designed for maximum throughput in the Hytale engine's data-heavy environment.

## Lifecycle & Ownership
- **Creation:** Instances are ephemeral and created for a single parsing task. They are typically instantiated via static factory methods like `fromPath` or `fromJsonString`. The most common pattern involves retrieving a reusable buffer from the thread-local pool and passing it to the reader's constructor, as demonstrated in the `readSync` utility method.

- **Scope:** The lifetime of a RawJsonReader is strictly bound to the scope of a single deserialization operation. Its `AutoCloseable` interface signals that it should be managed by a try-with-resources statement to guarantee resource cleanup.

- **Destruction:** The `close()` method must be called to release system resources. This action closes the underlying `Reader` (if one exists) and nullifies internal references to the buffer, making it eligible for garbage collection. The `closeAndTakeBuffer` variant provides a mechanism to return a potentially resized buffer to a pool for reuse.

**WARNING:** Failure to close a RawJsonReader that wraps a file stream will result in a file handle leak.

## Internal State & Concurrency
- **State:** A RawJsonReader is highly stateful and mutable. Its core state includes the character `buffer`, an integer `bufferIndex` acting as a cursor, and metadata for tracking line numbers and marked positions. The internal buffer can be dynamically resized and repopulated from the source stream during read operations.

- **Thread Safety:** **This class is fundamentally not thread-safe.** Its internal state is accessed and modified on every method call without any synchronization mechanisms. Sharing a single RawJsonReader instance across multiple threads will lead to data corruption and unpredictable behavior. It is designed exclusively for thread-confined usage.

The `ThreadLocal` buffer pool is a concurrency *pattern* that facilitates safe parallel processing. It ensures that each thread operates on its own isolated buffer and its own distinct RawJsonReader instance, thereby preventing data races.

## API Surface
The public API is designed for sequential token consumption. The following are key symbols in its contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| expect(char) | void | Amortized O(1) | Asserts the next character is the expected one and consumes it. Throws IOException on mismatch. |
| tryConsume(char) | boolean | Amortized O(1) | Consumes the next character if it matches. Returns true on success, false otherwise. |
| readString() | String | O(N) | Parses and returns a complete JSON string, handling all escape sequences. Allocates a new String. |
| readDoubleValue() | double | O(N) | Parses a numeric value using a high-performance parser. Does not allocate heap memory. |
| skipValue() | void | O(N) | Efficiently advances the stream past the next complete JSON value (object, array, string, etc.). |
| seekToKey(reader, search) | boolean | O(N) | A static utility to advance a reader to a specific key within a JSON object, skipping intermediate data. |
| close() | void | O(1) | Releases the underlying stream and buffer resources. Critical for preventing resource leaks. |

*Complexity Note: O(N) refers to the length of the token being read. Amortized O(1) operations may trigger an O(B) buffer fill operation, where B is the buffer size.*

## Integration Patterns

### Standard Usage
The canonical usage pattern involves a try-with-resources block, a `Codec`, and the thread-local buffer pool for file processing. This ensures performance and resource safety.

```java
// How a developer should normally use this
Path assetPath = Path.of("my_asset.json");
Codec<MyAsset> assetCodec = ...;
HytaleLogger logger = ...;

MyAsset asset = null;
try {
    asset = RawJsonReader.readSync(assetPath, assetCodec, logger);
} catch (IOException e) {
    // Handle parsing or I/O failure
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Sharing:** Never share a RawJsonReader instance between threads. Each thread must create its own.
- **Resource Leaks:** Do not instantiate a reader on a file stream without using a try-with-resources block or manually calling `close()` in a `finally` block.
- **Ignoring The Buffer Pool:** For frequent parsing operations, manually creating new `char[]` buffers for each reader is inefficient. The static `READ_BUFFER` pool should be used.
- **String Concatenation:** Avoid reading parts of the JSON into strings to manually parse them later. Use the built-in `read*Value` and `skipValue` methods to process the stream directly.

## Data Pipeline
RawJsonReader is a critical link in the chain that transforms serialized data on disk into live objects in the game engine.

> Flow:
> File System (Path) -> Files.newInputStream -> InputStreamReader -> **RawJsonReader** -> Codec.decodeJson -> Engine Domain Object


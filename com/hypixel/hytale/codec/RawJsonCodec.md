---
description: Architectural reference for RawJsonCodec
---

# RawJsonCodec<T>

**Package:** com.hypixel.hytale.codec
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface RawJsonCodec<T> {
```

## Architecture & Concepts
The RawJsonCodec interface defines a low-level contract for streaming deserialization of JSON data into a specific Java type T. It is a foundational component of the engine's data serialization pipeline, designed for high-performance, context-aware data conversion.

Unlike typical JSON libraries that parse data into an intermediate Document Object Model (DOM) tree, this interface operates directly on a RawJsonReader stream. This approach minimizes memory allocation and parsing overhead, making it suitable for performance-critical systems such as network packet decoding, asset loading, and configuration file processing.

The core architectural pattern is the separation of the data stream (RawJsonReader) from the decoding logic (the RawJsonCodec implementation). This allows for a single, optimized reader to be used with a multitude of type-specific decoders. The inclusion of an ExtraInfo parameter in the primary decode method enables context-sensitive deserialization, where the parsing logic can be altered based on external information provided by the calling system.

## Lifecycle & Ownership
As an interface, RawJsonCodec does not have a lifecycle of its own. The lifecycle and ownership semantics apply to the concrete classes that **implement** this interface.

- **Creation:** Implementations are typically created and registered with a central codec registry or service locator during application bootstrap. They are often designed as stateless singletons.
- **Scope:** The scope of a codec implementation is determined by its registration. Most codecs are application-scoped, living for the entire session. However, specialized codecs could be created with a more narrow scope, such as for a specific game world or player session.
- **Destruction:** Codec instances are typically garbage collected when the application or their owning registry is shut down. They do not manage unmanaged resources and generally do not require explicit destruction.

## Internal State & Concurrency
- **State:** The RawJsonCodec contract is inherently stateless. Implementations are **strongly encouraged** to be stateless and immutable to ensure they are reusable and thread-safe. A stateful implementation would be a significant deviation from the intended design and could introduce severe concurrency issues.
- **Thread Safety:** The interface itself provides no thread-safety guarantees. The thread safety of a codec is entirely dependent on its implementation. Stateless implementations are inherently thread-safe and can be shared across multiple threads.

**WARNING:** Developers must never assume a RawJsonCodec implementation is thread-safe. Always consult the documentation for the specific implementation or treat it as thread-unsafe unless it is explicitly designed as a stateless singleton.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decodeJson(RawJsonReader reader) | T | N/A | **Deprecated.** Legacy method for decoding without context. Should not be used in new code. |
| decodeJson(RawJsonReader var1, ExtraInfo var2) | T | O(N) | Deserializes the JSON stream from the reader into an object of type T. N is the number of tokens in the stream. Returns null if the JSON represents a null value. Throws IOException on parsing or I/O errors. |

## Integration Patterns

### Standard Usage
The codec is not intended to be invoked directly. A higher-level service, such as a CodecRegistry or an AssetManager, is responsible for selecting the appropriate codec and managing the deserialization process.

```java
// Hypothetical example of a registry using a codec
CodecRegistry registry = context.getService(CodecRegistry.class);
RawJsonReader reader = new RawJsonReader(someInputStream);
ExtraInfo context = new ExtraInfo(/* ... */);

// The registry finds the correct RawJsonCodec<MyObject> and invokes it
MyObject result = registry.decode(MyObject.class, reader, context);
```

### Anti-Patterns (Do NOT do this)
- **Calling Deprecated Methods:** Do not invoke the `decodeJson(RawJsonReader)` variant. It bypasses the context system and may lead to incorrect deserialization.
- **Ignoring Null Return:** The `decodeJson` method is annotated as Nullable. The JSON `null` literal will correctly deserialize to a Java `null`. Callers **must** perform a null check on the result.
- **Swallowing IOException:** This is a checked exception for a reason. I/O errors or malformed JSON are critical failures that must be handled or propagated up the call stack.
- **Assuming State:** Do not design systems that rely on a codec instance retaining state between calls. This violates the intended stateless nature of the pattern.

## Data Pipeline
RawJsonCodec is a key transformation step in the data ingestion pipeline. It converts a raw stream of characters into structured, type-safe Java objects that the rest of the engine can operate on.

> Flow:
> Raw Byte Stream (File, Network) -> Character Stream (e.g., InputStreamReader) -> RawJsonReader -> **RawJsonCodec<T> Implementation** -> Instantiated Java Object (T)


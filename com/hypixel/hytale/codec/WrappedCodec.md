---
description: Architectural reference for WrappedCodec
---

# WrappedCodec

**Package:** com.hypixel.hytale.codec
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface WrappedCodec<T> {
```

## Architecture & Concepts
The WrappedCodec interface establishes a formal contract for implementing the Decorator pattern within the Hytale codec system. Its primary architectural role is to provide a standardized mechanism for introspecting and unwrapping layers of codec functionality.

In a complex serialization pipeline, a core Codec responsible for a specific data type (e.g., PlayerState) might be wrapped by other Codecs that add cross-cutting concerns like compression, encryption, or versioning. While the top-level Codec is used for the actual encoding and decoding operations, other systems may need to identify the fundamental data type being processed.

WrappedCodec solves this by creating a navigable, chain-like structure. Systems can recursively call getChildCodec to "peel back" layers until they reach the innermost, non-wrapped Codec, thereby discovering the true nature of the data payload. This prevents tight coupling between high-level systems and the specific composition of the codec stack.

## Lifecycle & Ownership
- **Creation:** As an interface, WrappedCodec is not instantiated directly. Concrete classes that implement this interface are typically created and registered by a central CodecRegistry or a similar factory during application bootstrap.
- **Scope:** The contract itself is stateless and exists for the lifetime of the ClassLoader. Implementations are generally designed to be stateless singletons, persisting for the entire application session and shared across various network and serialization contexts.
- **Destruction:** Implementations are typically not destroyed and are garbage collected along with their ClassLoader on application shutdown.

## Internal State & Concurrency
- **State:** This interface is inherently stateless. It defines a contract for behavior, not data storage. Implementations are strongly expected to be immutable or effectively immutable. The returned child Codec should also follow these principles.
- **Thread Safety:** The contract is thread-safe. Any class implementing this interface **MUST** ensure that its implementation of getChildCodec is thread-safe. Given the expected singleton nature of codecs, this method should be free of side effects and simply return a pre-configured, final instance field.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChildCodec() | Codec<T> | O(1) | Retrieves the immediate, underlying Codec that this instance wraps. Returns the next link in the decorator chain. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to recursively unwrap a Codec instance to find its base type. This is critical for systems like packet loggers, debug tools, or dynamic dispatchers that need to operate on the core data type, irrespective of the serialization layers.

```java
// How a system should introspect a codec stack
public Codec<?> findBaseCodec(Codec<?> codec) {
    Codec<?> current = codec;
    while (current instanceof WrappedCodec) {
        // Cast and unwrap to the next layer
        current = ((WrappedCodec<?>) current).getChildCodec();
    }
    // The loop terminates when a non-wrapped, base codec is found
    return current;
}
```

### Anti-Patterns (Do NOT do this)
- **Shallow Inspection:** Systems that fail to check for and unwrap a WrappedCodec will operate with incomplete information. For example, a network analyzer might incorrectly classify a packet as "CompressedData" instead of its true "PlayerInventoryUpdate" type, leading to faulty diagnostics.
- **Stateful Implementations:** Implementing getChildCodec to dynamically generate or fetch a Codec is a severe anti-pattern. This method is expected to be a simple, high-performance accessor. Introducing I/O, computation, or mutable state will violate the architectural contract and introduce severe performance bottlenecks.

## Data Pipeline
WrappedCodec is not a direct participant in the data flow of serialization or deserialization. Instead, it is a meta-component used by other systems to understand the structure *of* the pipeline itself.

> **Introspection Flow:**
>
> System needing base type -> Receives top-level `Codec` -> Checks `instanceof WrappedCodec` -> Calls **`getChildCodec()`** -> Receives inner `Codec` -> Repeats until base `Codec` is found


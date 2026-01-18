---
description: Architectural reference for SetCodec
---

# SetCodec

**Package:** com.hypixel.hytale.codec.codecs.set
**Type:** Utility

## Definition
```java
// Signature
public class SetCodec<V, S extends Set<V>> implements Codec<Set<V>>, WrappedCodec<V> {
```

## Architecture & Concepts
The SetCodec is a composite, structural codec responsible for serializing and deserializing Java Set collections. It does not operate on the elements within the Set directly; instead, it delegates this responsibility to a child Codec instance provided during its construction. This follows the **Composite Pattern**, allowing complex object graphs to be built from simpler, reusable codec components.

Its primary architectural functions are:
1.  **Structure Mapping:** It maps the unordered, unique-element contract of a Java Set to a BSON or JSON array structure.
2.  **Constraint Enforcement:** During deserialization, it actively enforces the uniqueness constraint of a Set. If the incoming data stream contains duplicate elements, the decoding process will fail with a CodecException. This provides a critical data validation layer.
3.  **Implementation Abstraction:** Through the use of a `Supplier<S>`, the SetCodec is decoupled from any specific Set implementation like HashSet or LinkedHashSet. This allows consumers to control the memory and performance characteristics of the resulting collection.
4.  **Immutability Control:** A constructor flag allows the codec to produce unmodifiable Set instances upon decoding. This is a key feature for promoting thread safety and predictable state management in the broader application.

As a WrappedCodec, it provides access to its underlying element codec, enabling introspection and schema generation for the entire data structure.

## Lifecycle & Ownership
-   **Creation:** A SetCodec is not intended for direct instantiation by end-users. It is constructed and managed by a central codec registry or a factory system (e.g., a `Codecs` utility class). An instance is created for each unique `Set<V>` type that requires serialization, injecting the appropriate child codec for type `V`.
-   **Scope:** An instance of SetCodec is stateless and designed for reuse. Its lifetime is typically bound to the application's codec registry, meaning it persists for the entire session once created.
-   **Destruction:** The object is eligible for garbage collection when the codec registry is discarded, which usually occurs during application shutdown.

## Internal State & Concurrency
-   **State:** **Immutable and Stateless**. All internal fields (the child codec, the set supplier, and the unmodifiable flag) are final and set at construction. The SetCodec holds no state related to any specific encoding or decoding operation.
-   **Thread Safety:** **Conditionally Thread-Safe**. The SetCodec itself contains no locks and performs no mutable operations on its own state, making it safe for concurrent use. However, its overall thread safety is dependent on the thread safety of the child codec it wraps. If the child codec is thread-safe, then the SetCodec is also thread-safe.

## API Surface
The public API is focused on the core serialization contract defined by the Codec interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Set<V> | O(N) | Deserializes a BSON array into a Java Set. Throws CodecException on duplicate values or child decoding errors. |
| decodeJson(RawJsonReader, ExtraInfo) | Set<V> | O(N) | Deserializes a JSON array from a stream into a Java Set. Throws CodecException on duplicate values or child decoding errors. |
| encode(Set<V>, ExtraInfo) | BsonValue | O(N) | Serializes a Java Set into a BSON array by delegating element encoding to the child codec. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a schema representation, correctly identifying the type as an array with unique items. |
| getChildCodec() | Codec<V> | O(1) | Provides access to the wrapped codec responsible for the Set's elements. |

*N = number of elements in the Set or array.*

## Integration Patterns

### Standard Usage
The SetCodec should be obtained from a central factory or registry, which handles the injection of the correct child codec.

```java
// Assume a factory 'Codecs' exists for creating standard codec instances.
// This is the correct way to obtain a codec for a Set of Strings.
Codec<Set<String>> stringSetCodec = Codecs.setOf(Codecs.STRING);

// Use the codec to process data
Set<String> mySet = new HashSet<>();
mySet.add("alpha");
mySet.add("beta");

BsonValue encodedData = stringSetCodec.encode(mySet, ExtraInfo.NONE);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SetCodec(...)`. This bypasses the framework's management and caching of codec instances and can lead to unnecessary object creation. Always use the provided factories.
-   **Incorrect Child Codec:** Passing a codec of the wrong type (e.g., a `Codec<Integer>` for a `Set<String>`) will lead to runtime ClassCastExceptions or serialization errors. The type parameter `V` must be consistent.
-   **Ignoring Uniqueness Errors:** The codec will throw an exception if the source data contains duplicates. Do not catch and ignore this exception, as it indicates a data integrity violation. The source data must be corrected.

## Data Pipeline

The SetCodec acts as a specific node in the overall serialization and deserialization pipeline.

**Deserialization Flow:**
> BSON Array -> **SetCodec.decode** -> (Iteration) -> Child Codec.decode -> Set::add -> Java Set Object

**Serialization Flow:**
> Java Set Object -> **SetCodec.encode** -> (Iteration) -> Child Codec.encode -> BsonArray::add -> BSON Array


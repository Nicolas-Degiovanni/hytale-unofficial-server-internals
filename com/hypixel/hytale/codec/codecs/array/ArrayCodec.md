---
description: Architectural reference for ArrayCodec
---

# ArrayCodec<T>

**Package:** com.hypixel.hytale.codec.codecs.array
**Type:** Transient Component

## Definition
```java
// Signature
public class ArrayCodec<T> implements Codec<T[]>, RawJsonCodec<T[]>, WrappedCodec<T> {
```

## Architecture & Concepts
The ArrayCodec is a fundamental component of the Hytale serialization framework, responsible for encoding and decoding native Java arrays (T[]). It embodies the **Composite Pattern**, acting as a specialized container codec that does not understand the internal structure of its elements. Instead, it delegates the serialization of each individual element to a "child" Codec provided during its construction.

Its primary role is to manage the array structure itself—the BsonArray envelope for BSON serialization or the square bracket syntax `[]` for JSON—while iterating through the elements and invoking the appropriate child codec.

A key architectural feature is the use of an `IntFunction<T[]> arrayConstructor`. Due to Java's type erasure, it is not possible to instantiate a generic array directly (e.g., `new T[size]`). This function provides a type-safe factory mechanism, supplied by the developer, to correctly allocate the destination array at runtime. This design keeps the serialization logic generic and reusable while ensuring type safety at the point of use.

The ArrayCodec is a foundational building block, designed to be composed within more complex object schemas rather than used in isolation.

### Lifecycle & Ownership
-   **Creation:** An ArrayCodec is instantiated directly via its constructor, typically during the static initialization of a larger schema definition or codec registry. The static factory method `ofBuilderCodec` provides a convenient alternative for codecs based on the BuilderCodec pattern. It is not managed by a dependency injection container.
-   **Scope:** The lifecycle of an ArrayCodec instance is bound to the lifecycle of the schema or registry that contains it. In most scenarios, this means it is created once at application startup and persists for the entire session.
-   - **WARNING:** These objects are designed for reuse. Creating new ArrayCodec instances for each serialization operation is highly inefficient and considered an anti-pattern.
-   **Destruction:** Instances are subject to standard Java garbage collection when the containing schema is no longer referenced, typically upon server or client shutdown.

## Internal State & Concurrency
-   **State:** The ArrayCodec is stateful, but its state is primarily immutable configuration established at construction time (the child codec, the array constructor, and the default value supplier).
    -   It contains one piece of mutable, lazily-initialized state: a cached `emptyArray` instance. This is a performance optimization to avoid re-allocating zero-length arrays.
    -   The `metadata` list is also mutable via the `metadata` method. This implies a configuration phase followed by a usage phase.
-   **Thread Safety:** The class is **conditionally thread-safe**.
    -   The primary `encode` and `decode` methods are re-entrant and safe for concurrent use, provided the instance has been fully configured beforehand. They operate on local variables and passed-in parameters.
    -   **WARNING:** Configuration methods, specifically `metadata`, are **not thread-safe**. Modifying an ArrayCodec instance after it has been shared across threads will lead to unpredictable behavior. All configuration must be completed in a single-threaded context before the codec is used for active serialization.
    -   The lazy initialization of the `emptyArray` field presents a benign check-then-act race condition. While this could result in multiple threads creating an empty array, the outcome is harmless as the final state is identical.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | T[] | O(N) | Deserializes a BsonArray into a new Java array of type T. N is the element count. |
| encode(array, extraInfo) | BsonValue | O(N) | Serializes a Java array into a BsonArray. N is the element count. |
| decodeJson(reader, extraInfo) | T[] | O(N) | Deserializes a JSON array from a raw character stream into a new Java array. |
| getChildCodec() | Codec<T> | O(1) | Returns the inner codec used for serializing individual array elements. |
| metadata(metadata) | ArrayCodec<T> | O(1) | Appends schema metadata to the codec. Not thread-safe. Returns self for chaining. |
| toSchema(context) | Schema | O(M) | Generates a formal schema definition for this array type. M is the metadata count. |
| ofBuilderCodec(codec, constructor) | ArrayCodec<T> | O(1) | Static factory for creating an ArrayCodec from a BuilderCodec. |

## Integration Patterns

### Standard Usage
An ArrayCodec is almost never invoked directly. It is composed into a larger schema for an object that contains an array field. The framework's top-level serializer then discovers and delegates to the ArrayCodec when it encounters that field.

```java
// Example: Defining a codec for a class that contains an array of strings.
// The ArrayCodec is constructed once and assigned to a static final field.

public static final Codec<String[]> STRING_ARRAY_CODEC = new ArrayCodec<>(
    Codec.STRING, // The child codec for elements
    String[]::new   // The array constructor function
);

public static final Codec<MyObject> MY_OBJECT_CODEC = BuilderCodec.of(
    MyObject::new,
    "names", MyObject::getNames, STRING_ARRAY_CODEC,
    // ... other fields
);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Do not call the `metadata` method on a shared codec instance from multiple threads or after the initialization phase. This is a critical race condition.
-   **Per-Operation Instantiation:** Do not create a `new ArrayCodec` for every array you need to serialize. They are heavyweight objects designed to be singletons for a given type.
-   **Incorrect Type Usage:** This codec is strictly for `T[]`. Do not attempt to use it for `List<T>`, `Set<T>`, or other Java Collections. The framework provides different codecs (e.g., ListCodec) for those types.

## Data Pipeline

The ArrayCodec acts as a marshalling stage in the overall serialization pipeline.

> **Decoding Flow:**
> BsonValue (BsonArray) -> **ArrayCodec.decode** -> Loop -> Invoke `childCodec.decode` on each element -> Populate new `T[]` -> Return `T[]`

> **Encoding Flow:**
> `T[]` (Java Array) -> **ArrayCodec.encode** -> Create BsonArray -> Loop -> Invoke `childCodec.encode` on each element -> Add to BsonArray -> Return BsonValue


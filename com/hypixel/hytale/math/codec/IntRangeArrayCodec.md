---
description: Architectural reference for IntRangeArrayCodec
---

# IntRangeArrayCodec

**Package:** com.hypixel.hytale.math.codec
**Type:** Transient Utility

## Definition
```java
// Signature
public class IntRangeArrayCodec implements Codec<IntRange>, ValidatableCodec<IntRange> {
```

## Architecture & Concepts
The IntRangeArrayCodec is a specialized, stateless component within the Hytale data serialization framework. Its sole responsibility is to act as a bidirectional translator for the **IntRange** data type, converting it to and from a concrete BSON or JSON representation.

Architecturally, it is a "leaf" node in the codec system. It handles a primitive, well-defined data structure and has no dependencies on other complex codecs. The chosen on-disk and network format is a fixed-size, two-element array of integers: `[inclusiveMin, inclusiveMax]`.

This class embodies the principle of self-contained data definition. By implementing both **Codec** and **ValidatableCodec**, it provides three critical functions for the IntRange type:
1.  **Serialization:** Converting the in-memory IntRange object to a BSON or JSON array.
2.  **Deserialization:** Parsing a BSON or JSON array back into a live IntRange object.
3.  **Validation:** Enforcing the logical constraint that the minimum value cannot exceed the maximum value.

This encapsulation allows the broader engine to handle complex data assets without needing to understand the specific binary layout of every nested data type.

### Lifecycle & Ownership
-   **Creation:** Instances of IntRangeArrayCodec are not intended for direct user creation. They are instantiated by a central **CodecRegistry** or a similar factory during the engine's bootstrap phase. This typically happens once when the engine registers all known data types and their corresponding serializers.
-   **Scope:** The object is stateless and reusable. A single instance is typically shared across the entire application and lives for the duration of the process. Its lifecycle is bound to the lifecycle of the parent CodecRegistry.
-   **Destruction:** The object is eligible for garbage collection only when the CodecRegistry that holds a reference to it is destroyed, which usually occurs during a clean application shutdown.

## Internal State & Concurrency
-   **State:** **Stateless and Immutable**. The IntRangeArrayCodec contains no member fields and holds no data between method invocations. All operations are pure functions that depend only on their input arguments.
-   **Thread Safety:** **Fully Thread-Safe**. Due to its stateless nature, a single instance can be safely and concurrently invoked by multiple threads without any need for external locking or synchronization. This is a critical design feature for high-throughput systems, such as parallel asset loading or concurrent network packet processing.

## API Surface
The public API provides the core functions for serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | IntRange | O(1) | Deserializes a BsonValue (expected to be a BsonArray) into an IntRange object. Throws if the BsonValue is not a two-element numeric array. |
| encode(t, extraInfo) | BsonValue | O(1) | Serializes an IntRange object into a new BsonArray containing two BsonDouble values. |
| decodeJson(reader, extraInfo) | IntRange | O(1) | Deserializes a raw JSON stream representing an array (e.g., `[10, 20]`) into an IntRange object. Manages its own parsing state. |
| toSchema(context) | Schema | O(1) | Generates a formal schema definition describing the data as a fixed-size, two-element integer array. Used for tooling and automated validation. |
| validate(range, extraInfo) | void | O(1) | Executes logical validation on an IntRange object, ensuring `min <= max`. Reports failures to the ValidationResults context. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. Instead, it is used implicitly by the higher-level codec system. The engine retrieves the appropriate codec from a registry based on the target data type and delegates the operation.

```java
// Hypothetical usage within a higher-level data manager
// The developer works with high-level objects, not specific codecs.

// 1. A BSON document is received from a file or network
BsonDocument data = BsonDocument.parse("{ 'damageRange': [10, 25] }");

// 2. The engine's CodecService is asked to decode the document
// The service internally looks up the codec for the 'damageRange' field.
// It finds and uses IntRangeArrayCodec behind the scenes.
WeaponConfig config = codecService.decode(data, WeaponConfig.class);

// 3. The resulting object is a fully materialized Java object
IntRange damage = config.getDamageRange(); // Returns an IntRange(10, 25)
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new IntRangeArrayCodec()`. The codec framework is responsible for the lifecycle of codec instances. Manually creating instances can bypass registration and caching, leading to degraded performance and unpredictable behavior.
-   **Invalid Input:** Passing a BSON document, a non-array BSON value, or an array with a non-2 element count to the `decode` method will result in an immediate runtime exception. The codec expects its input to be structurally correct.

## Data Pipeline
The IntRangeArrayCodec serves as a specific, typed step in a larger data transformation pipeline.

**Deserialization (Loading)**
> Flow:
> Raw Bytes (File/Network) -> BSON Parser -> BsonArray `[10, 20]` -> **IntRangeArrayCodec.decode** -> In-Memory `IntRange` Object -> Game Logic

**Serialization (Saving)**
> Flow:
> Game Logic -> In-Memory `IntRange` Object -> **IntRangeArrayCodec.encode** -> BsonArray `[10, 20]` -> BSON Emitter -> Raw Bytes (File/Network)


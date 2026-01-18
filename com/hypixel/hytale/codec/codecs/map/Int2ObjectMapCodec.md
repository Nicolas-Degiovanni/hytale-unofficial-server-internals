---
description: Architectural reference for Int2ObjectMapCodec
---

# Int2ObjectMapCodec<T>

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Utility

## Definition
```java
// Signature
public class Int2ObjectMapCodec<T> implements Codec<Int2ObjectMap<T>>, WrappedCodec<T> {
```

## Architecture & Concepts
The Int2ObjectMapCodec is a specialized, high-performance component within the engine's serialization framework. Its primary function is to translate between in-memory `Int2ObjectMap` collections and their serialized representations in BSON or JSON formats. This codec is critical for performance-sensitive data structures where using a primitive `int` as a map key avoids the overhead of Java's `Integer` boxing.

The design of this class is centered on the principle of **composition**. It is a generic wrapper that does not intrinsically understand how to process the map's values of type `T`. Instead, it delegates this responsibility to a "child" codec, an instance of `Codec<T>`, which is provided during construction. This composite pattern makes the Int2ObjectMapCodec extremely reusable across any data model that employs integer-keyed maps.

Key architectural features include:
*   **Pluggable Map Implementation:** A `Supplier` function is provided at construction to instantiate the target map. This decouples the codec from a specific `Int2ObjectMap` implementation, allowing developers to choose the most appropriate one (e.g., `Int2ObjectOpenHashMap`, `Int2ObjectArrayMap`) for their use case.
*   **Immutability Enforcement:** An optional `unmodifiable` flag can be set. When enabled, the codec wraps the newly created map from a `decode` operation in an unmodifiable view. This is a powerful tool for enforcing read-only contracts and preventing unintended state mutations in downstream systems.
*   **Dual-Format Support:** It provides distinct, optimized paths for handling both BSON documents and raw JSON streams, ensuring broad compatibility with different data interchange scenarios.

## Lifecycle & Ownership
- **Creation:** Instances are not managed by a central service locator. They are manually instantiated by developers when composing more complex codecs. For example, a `WorldChunkCodec` might create and hold a reference to an `Int2ObjectMapCodec` to handle a map of block entity IDs to block entity data.
- **Scope:** The lifecycle of an Int2ObjectMapCodec instance is bound to the parent object that creates it, typically another codec. These objects are designed to be lightweight and are intended to be created once during application initialization and reused for the entire session.
- **Destruction:** The object is eligible for garbage collection when the parent codec or configuration that holds its reference is unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The Int2ObjectMapCodec is **effectively immutable**. Its internal fields, including the child `valueCodec` and the map `supplier`, are final and set only at construction time. The `encode` and `decode` methods are pure functions that operate exclusively on their inputs and do not modify any internal state.
- **Thread Safety:** This class is **thread-safe**. Due to its immutable nature, a single instance can be safely shared and invoked by multiple threads concurrently without risk of race conditions or data corruption. This guarantee is contingent on the provided child `valueCodec` also being thread-safe, which is the standard contract for all core engine codecs.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Int2ObjectMap<T> | O(N * M) | Deserializes a BSON document into a map. N is the number of entries; M is the complexity of the child value decoder. |
| encode(map, extraInfo) | BsonValue | O(N * P) | Serializes a map into a BSON document. N is the number of entries; P is the complexity of the child value encoder. |
| decodeJson(reader, extraInfo) | Int2ObjectMap<T> | O(N * M) | Deserializes a JSON object from a raw stream into a map. Performance is highly dependent on the underlying stream. |
| getChildCodec() | Codec<T> | O(1) | Returns the composed codec used for serializing map values. |
| toSchema(context) | Schema | O(1) | Generates a data schema representing the map structure for validation or documentation purposes. |

## Integration Patterns

### Standard Usage
The codec is designed to be composed within other, more complex codecs. It is never used in isolation. The standard pattern involves creating an instance to handle a specific field during the definition of a parent object's codec.

```java
// Example: Defining a codec for a player's inventory, which is a map of slot IDs to ItemStacks.
// Assume an existing 'ITEM_STACK_CODEC' is defined elsewhere.

public static final Codec<PlayerInventory> INVENTORY_CODEC = new ObjectCodec.Builder<PlayerInventory>()
    .field(
        "items",
        // Instantiate the map codec, providing the child codec for ItemStacks
        // and a supplier for the desired map implementation.
        new Int2ObjectMapCodec<>(ITEM_STACK_CODEC, Int2ObjectOpenHashMap::new),
        PlayerInventory::getItems
    )
    .build(PlayerInventory::new);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Suppliers:** The `Supplier` provided to the constructor must *always* return a new, empty map instance. Using a supplier that returns a shared or singleton map instance will lead to severe data corruption, as decode operations from multiple sources will write to the same object.
- **Modifying Unmodifiable Maps:** If the codec is constructed with `unmodifiable = true`, do not attempt to modify the map returned by a `decode` operation. Doing so will result in an `UnsupportedOperationException` at runtime. The unmodifiable flag exists to enforce a read-only contract.

## Data Pipeline

### Decode Flow
The data pipeline for deserialization transforms a serialized document into a rich, in-memory Java object.

> Flow:
> BsonDocument -> **Int2ObjectMapCodec**.decode() -> Iterate BSON entries -> (Integer.parseInt(key), valueCodec.decode(value)) -> map.put() -> Int2ObjectMap<T>

### Encode Flow
The encoding pipeline is the reverse, converting the in-memory map into a BSON document suitable for network transmission or storage.

> Flow:
> Int2ObjectMap<T> -> **Int2ObjectMapCodec**.encode() -> Iterate map entries -> (Integer.toString(key), valueCodec.encode(value)) -> BsonDocument.put() -> BsonDocument


---
description: Architectural reference for WeightedMapCodec
---

# WeightedMapCodec

**Package:** com.hypixel.hytale.server.core.codec
**Type:** Transient / Utility

## Definition
```java
// Signature
public class WeightedMapCodec<T extends IWeightedElement> implements Codec<IWeightedMap<T>>, WrappedCodec<T> {
```

## Architecture & Concepts
The WeightedMapCodec is a specialized serialization component designed to encode and decode `IWeightedMap` data structures. This data structure is common in game development for implementing weighted random selections, such as loot tables, spawn pools, or procedural generation choices.

Architecturally, this class follows the **Wrapper** or **Decorator** pattern. It does not handle the serialization of the map's individual elements (`T`). Instead, it is constructed with a child `Codec<T>` which it delegates to for processing each element. This design promotes separation of concerns and reusability; a single `WeightedMapCodec` can be parameterized to handle any element type, provided a corresponding element codec exists.

It serves as a bridge between the logical in-memory representation of a weighted map and its physical representation in data files. The standard serialized format is a simple array (BSON or JSON), where each object in the array is an element of the map. The weight of each element is not stored separately by this codec; it is assumed to be an intrinsic property of the element itself, accessible via the `IWeightedElement` interface contract.

## Lifecycle & Ownership
-   **Creation:** WeightedMapCodec instances are not intended for direct instantiation by end-users. They are typically created and managed by a central codec registry or a factory system during application bootstrap. The parent system is responsible for supplying the necessary dependencies: the child `Codec<T>` for the element type and an empty array of `T` for efficient `WeightedMap` builder initialization.

-   **Scope:** An instance of WeightedMapCodec is stateless and its lifetime is tied to the codec registry that owns it. It can be safely treated as a singleton for a given generic type `T` and will persist for the entire application session.

-   **Destruction:** The object is eligible for garbage collection when its owning codec registry is unloaded, typically during a server shutdown or a major context change.

## Internal State & Concurrency
-   **State:** This class is **immutable**. Its internal fields, `codec` and `emptyKeys`, are assigned at construction and never change. It holds no session-specific or call-specific state, making its behavior deterministic.

-   **Thread Safety:** The WeightedMapCodec is **thread-safe**. Due to its immutable and stateless nature, a single instance can be safely used by multiple threads concurrently to perform encoding and decoding operations without locks or synchronization. Thread safety of the overall operation depends on the thread safety of the supplied child `codec`, which is a standard expectation within the Hytale codec framework.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChildCodec() | Codec<T> | O(1) | Returns the wrapped codec responsible for element serialization. |
| decode(bsonValue, extraInfo) | IWeightedMap<T> | O(N) | Deserializes a BSON array into a new IWeightedMap instance. N is the number of elements. |
| encode(map, extraInfo) | BsonValue | O(N) | Serializes an IWeightedMap into a BSON array. N is the number of elements. |
| decodeJson(reader, extraInfo) | IWeightedMap<T> | O(N) | Deserializes a JSON array from a raw reader into a new IWeightedMap instance. |
| toSchema(context) | Schema | O(1) | Generates a schema definition representing a weighted map as an array of its elements. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. It is automatically invoked by the framework's primary serialization engine when a field of type `IWeightedMap` is encountered. The framework is responsible for locating and providing the correct child codec for the map's generic type.

The following example shows a hypothetical parent object being decoded, which would internally trigger the WeightedMapCodec.

```java
// Assume LootTable has a field: IWeightedMap<LootItem> items;
// The framework's CodecRegistry is pre-configured with a
// WeightedMapCodec<LootItem> instance.

Codec<LootTable> lootTableCodec = CodecRegistry.get(LootTable.class);

// This call will internally find and use WeightedMapCodec to decode the 'items' field
LootTable table = lootTableCodec.decode(bsonData, ExtraInfo.NONE);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new WeightedMapCodec(...)`. This bypasses the central codec management system, leading to inconsistent serialization behavior and potential runtime errors if the correct dependencies are not provided.

-   **Stateful Child Codec:** Providing a child `Codec<T>` that is not thread-safe or that maintains state between calls can violate the concurrency guarantees of the WeightedMapCodec and lead to unpredictable behavior under load.

-   **Data Mismatch:** Attempting to decode data that is not an array will result in a `CodecException`. The data source must be a BSON or JSON array where each element is a valid representation of type `T`.

## Data Pipeline

The WeightedMapCodec acts as a specific transformation step within a larger data serialization pipeline.

> **Decoding Flow:**
> BSON Array -> BsonValue Iteration -> **WeightedMapCodec.decode** -> (For each element: `childCodec.decode`) -> WeightedMap.Builder -> In-memory `IWeightedMap<T>` Object

> **Encoding Flow:**
> In-memory `IWeightedMap<T>` Object -> **WeightedMapCodec.encode** -> (For each element: `childCodec.encode`) -> BsonArray Construction -> BSON Array


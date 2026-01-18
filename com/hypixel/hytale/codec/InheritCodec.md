---
description: Architectural reference for InheritCodec
---

# InheritCodec<T>

**Package:** com.hypixel.hytale.codec
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface InheritCodec<T> extends Codec<T>, RawJsonInheritCodec<T> {
```

## Architecture & Concepts
The InheritCodec interface defines a specialized contract for deserialization within Hytale's data and asset management systems. It extends the base Codec interface, adding a critical capability: **deserialization with inheritance**. This pattern is fundamental for systems that rely on hierarchical data, such as asset prefabs, entity variants, or layered configurations.

Where a standard Codec typically creates an object from a data source from scratch, an InheritCodec is designed to merge data from a BsonDocument onto an *existing* parent object. This allows for the creation of derivative assets that override or extend properties from a base template. For example, a *GoblinShaman* asset can inherit all properties from a base *Goblin* asset and then apply its own unique properties (e.g., a staff, magical abilities) from a separate data file.

This interface is a cornerstone of the asset pipeline, enabling efficient data reuse and reducing data duplication across the game's content files. It acts as the logical engine for materializing complex, layered objects from the BSON data format used for game assets.

### Lifecycle & Ownership
- **Creation:** Implementations of InheritCodec are typically stateless, singleton-like objects. They are instantiated once during the application's bootstrap phase and registered with a central CodecRegistry or a similar service locator.
- **Scope:** A registered codec instance persists for the entire application session. It is a long-lived, shared utility.
- **Destruction:** Codec instances are cleaned up only when the application shuts down and the CodecRegistry is cleared. There is no per-object or per-use lifecycle.

## Internal State & Concurrency
- **State:** As an interface, InheritCodec is stateless. All concrete implementations of this interface **must** be designed to be stateless. All data required for an operation is passed explicitly through method arguments (the BsonDocument, the parent object, and the ExtraInfo context). Storing session-specific or per-call state within a codec is a severe anti-pattern.
- **Thread Safety:** Implementations are expected to be fully thread-safe. The stateless design facilitates this, allowing a single codec instance to be safely and concurrently invoked by multiple worker threads, such as those in a parallel asset loading system. No internal locking should be necessary if the contract is followed correctly.

## API Surface
The public contract is focused on two methods for applying inherited data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decodeAndInherit(doc, parent, info) | T | O(N) | Deserializes the BsonDocument and merges its data onto a copy of the provided parent object. Returns a **new** instance representing the final, merged state. May return null on failure. |
| decodeAndInherit(doc, parent, dest, info) | void | O(N) | A performance-oriented variant. Deserializes the BsonDocument and merges its data onto the *destination* object, using the parent for reference. This method modifies the destination object in-place. |

**Warning:** The complexity is O(N), where N is the number of fields in the BsonDocument. Performance of codec implementations is critical, as they are on the hot path for all asset loading.

## Integration Patterns

### Standard Usage
Developers do not typically invoke an InheritCodec directly. The engine's serialization or asset management services abstract this process. The system internally selects the appropriate codec from a registry and uses it to materialize an asset.

```java
// Conceptual example of how the AssetManager might use a codec
Asset parentAsset = assetCache.get("hytale:goblin");
BsonDocument variantData = loadBson("hytale:goblin_shaman.json");

// The manager finds the correct InheritCodec for the Asset type
InheritCodec<Asset> codec = codecRegistry.getInheritCodec(Asset.class);

// It uses the codec to create the final, merged asset
Asset shamanAsset = codec.decodeAndInherit(variantData, parentAsset, extraInfo);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not store any per-call data as fields within a codec implementation. This will immediately break thread safety and cause unpredictable behavior in the asset loader.
- **Incorrect Interface Usage:** Do not implement InheritCodec for data structures that are not hierarchical. For simple, self-contained objects, implement the base Codec interface instead. Using InheritCodec implies a specific data relationship that must be honored.
- **Ignoring Parent Data:** An implementation of `decodeAndInherit` must correctly consult the parent object when a field is absent in the BsonDocument. Failing to do so violates the "inheritance" contract.

## Data Pipeline
The InheritCodec sits at a critical juncture in the asset loading pipeline, responsible for the final materialization of an object from its constituent data parts.

> Flow:
> Asset File (JSON/BSON) on Disk -> File I/O -> BSON Parser -> BsonDocument -> Asset Manager -> **InheritCodec** -> Final Game Object in Memory


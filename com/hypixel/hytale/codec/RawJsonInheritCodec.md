---
description: Architectural reference for RawJsonInheritCodec
---

# RawJsonInheritCodec

**Package:** com.hypixel.hytale.codec
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface RawJsonInheritCodec<T> extends RawJsonCodec<T> {
```

## Architecture & Concepts
The RawJsonInheritCodec interface is a specialized contract for deserialization within Hytale's asset system. It extends the base RawJsonCodec to introduce the critical concept of **data inheritance**. This mechanism is fundamental to the engine's content variation and modding capabilities.

Where a standard codec deserializes a complete object from a JSON source, an implementation of RawJsonInheritCodec merges a JSON source *onto an existing parent object*. This allows content creators to define new assets as deltas or overrides of a base template. For example, a "burning_sword" asset might inherit all properties from a "base_sword" and only override its texture and particle effect properties in its own JSON file.

This interface is the bridge that performs this merge operation, combining the state of a fully-realized parent object with the partial data from a child's JSON definition. It is a cornerstone of the engine's "Composition over Inheritance" strategy for game assets.

### Lifecycle & Ownership
As an interface, RawJsonInheritCodec defines a contract and has no lifecycle itself. Its implementations, however, follow a strict lifecycle.

- **Creation:** Implementations are discovered and instantiated a single time by the CodecRegistry during application bootstrap. They are typically stateless utility classes.
- **Scope:** An implementation's instance is a singleton that persists for the entire application session. It is registered in a central lookup service and retrieved by the asset loading pipeline as needed.
- **Destruction:** The instance is discarded during application shutdown when the CodecRegistry is cleared.

## Internal State & Concurrency
- **State:** Implementations of this interface are **required** to be stateless. They must not contain any mutable fields that persist between calls. They function as pure data transformers.
- **Thread Safety:** Implementations **must be** thread-safe. The Hytale asset pipeline is highly concurrent and may invoke the same codec implementation from multiple worker threads simultaneously to load different assets. Storing any per-call state within the codec will lead to critical race conditions and data corruption.

## API Surface
The API defines two distinct modes of inheritance: one that creates a new object and one that mutates an existing one.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decodeAndInheritJson(reader, parent, info) | T | O(N) | **Creation Mode.** Deserializes JSON from the reader and merges it with the state of the *parent* object to produce a **new** object instance. Returns null if the JSON stream is empty or represents a null value. |
| decodeAndInheritJson(reader, parent, target, info) | void | O(N) | **Mutation Mode.** Deserializes JSON from the reader and merges it with the state of the *parent* object, applying the final result to the mutable *target* object. This is a performance optimization to avoid allocations when an object instance already exists. |

## Integration Patterns

### Standard Usage
This interface is not intended for direct use by gameplay or UI logic. It is an internal component of the asset loading system. The system orchestrates the loading of a parent asset, retrieves the appropriate codec, and then invokes it to produce the final, merged child asset.

```java
// Conceptual example from within the asset loading system

// 1. The system has already loaded the parent object
Model parentModel = assetCache.get("hytale:base_sword");

// 2. The system gets the codec for the Model class
RawJsonInheritCodec<Model> codec = CodecRegistry.getInheritCodec(Model.class);

// 3. A reader is prepared for the child asset's JSON file
RawJsonReader childJsonReader = getReaderFor("hytale:burning_sword.json");

// 4. The codec is invoked to produce the final, merged asset
Model burningSword = codec.decodeAndInheritJson(childJsonReader, parentModel, extraInfo);

// 5. The new asset is now ready for use
assetCache.put("hytale:burning_sword", burningSword);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Never store any instance variables in a class that implements this interface. The asset loader assumes codecs are stateless and thread-safe. Storing state will cause unpredictable behavior and crashes during parallel asset loading.
- **Direct Invocation:** Avoid looking up and invoking these codecs from game code. Always use the high-level AssetManager. The manager handles crucial steps like dependency resolution, caching, and asynchronous loading that you would otherwise have to replicate.

## Data Pipeline
The RawJsonInheritCodec is a key processing stage in the asset data pipeline. It is responsible for the logic of merging hierarchical data definitions.

> Flow:
> Asset Request -> AssetManager -> Parent Asset Resolution -> **RawJsonInheritCodec** -> Merged Asset Object -> Asset Cache -> Requester



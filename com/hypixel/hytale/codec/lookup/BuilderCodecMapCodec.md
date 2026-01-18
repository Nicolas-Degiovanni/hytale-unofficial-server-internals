---
description: Architectural reference for BuilderCodecMapCodec
---

# BuilderCodecMapCodec<T>

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Transient

## Definition
```java
// Signature
public class BuilderCodecMapCodec<T> extends StringCodecMapCodec<T, BuilderCodec<? extends T>> {
```

## Architecture & Concepts
The BuilderCodecMapCodec is a specialized dispatcher within the engine's serialization framework. It is designed to handle polymorphic data structures, where a string identifier within a data stream determines the concrete type of the object to be decoded.

This class acts as a registry, mapping string keys (e.g., "hytale:zone", "hytale:player_state") to specific BuilderCodec instances. The core lookup and dispatch logic is inherited from its parent, StringCodecMapCodec. When decoding, the parent reads a string identifier from the input, and this class uses that string to find the corresponding BuilderCodec in its internal map. That specific codec is then delegated to for decoding the rest ofthe object's data.

The primary specialization of this class is its understanding of the BuilderCodec contract. A BuilderCodec not only knows how to encode and decode a type but also how to construct a default instance of that type. The BuilderCodecMapCodec exposes this capability through its `getDefault` method, which provides a robust fallback mechanism for cases of missing or malformed data.

## Lifecycle & Ownership
- **Creation:** Instances are typically created during the bootstrap phase of the application or a major subsystem. They are constructed by a higher-level registry or a configuration service that defines the serialization rules for a particular data model (e.g., network packets, world save files).
- **Scope:** The lifetime of a BuilderCodecMapCodec is tied to its containing registry. For globally applicable codecs, such as those for network protocols, the instance persists for the entire application session. For context-specific codecs, such as for a temporary asset format, it may be short-lived.
- **Destruction:** The object is eligible for garbage collection when its owning registry is dereferenced, typically upon server or client shutdown.

## Internal State & Concurrency
- **State:** This class is stateful. It inherits and manages a map of string identifiers to BuilderCodec instances. This map is populated during an initial configuration phase and is intended to be treated as immutable thereafter.

- **Thread Safety:** **This class is not thread-safe for mutation.** The underlying map is not concurrent. All registration of codecs into the map must be completed in a single-threaded context before the codec is used for any encoding or decoding operations. Once populated, the instance is safe for concurrent reads, a common pattern for dispatchers and registries in the engine.

    **WARNING:** Modifying the internal codec map from multiple threads or after the codec is in active use will lead to race conditions and non-deterministic serialization behavior.

## API Surface
The primary API is inherited from the codec hierarchy for encoding and decoding. The method defined here provides specialized default-value handling.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDefault() | T | O(1) | Retrieves the registered default codec and invokes its `getDefaultValue` method. Throws a runtime exception if no default codec is registered. |

## Integration Patterns

### Standard Usage
A BuilderCodecMapCodec is configured once with all possible subtypes and then used as part of a larger object's serialization definition.

```java
// Example: Defining a codec for a generic "Entity" container
// 1. Create the dispatcher codec
BuilderCodecMapCodec<Entity> entityCodec = new BuilderCodecMapCodec<>("entityType");

// 2. Register the specific entity codecs during bootstrap
entityCodec.register("hytale:mob", new MobCodec());
entityCodec.register("hytale:player", new PlayerCodec());
entityCodec.setDefault(new MobCodec()); // Fallback

// 3. Use it in a higher-level codec
public static final Codec<WorldChunk> CHUNK_CODEC = new ObjectCodec.Builder<WorldChunk>()
    .field("entities", new ListCodec<>(entityCodec))
    .build(WorldChunk::new);
```

### Anti-Patterns (Do NOT do this)
- **Late Registration:** Do not register new codecs after the BuilderCodecMapCodec has been passed to other systems for active use. All registrations must occur during a controlled, single-threaded initialization phase.
- **Missing Default:** If the codec is constructed with `allowDefault` set to true, failing to register a default codec via `setDefault` will cause `getDefault` to fail at runtime. Ensure a default is always provided if the contract allows for it.

## Data Pipeline
The BuilderCodecMapCodec acts as a routing mechanism during the deserialization process. It does not transform data itself but rather selects the appropriate component to continue the pipeline.

> **Deserialization Flow:**
> Serialized Byte Stream -> Parent Deserializer -> String Identifier ("hytale:player") is read -> **BuilderCodecMapCodec** (Performs map lookup) -> PlayerCodec (Selected) -> Deserialized Player Object


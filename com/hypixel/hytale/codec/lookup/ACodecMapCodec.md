---
description: Architectural reference for ACodecMapCodec
---

# ACodecMapCodec

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Component

## Definition
```java
// Signature
public abstract class ACodecMapCodec<K, T, C extends Codec<? extends T>> implements Codec<T>, ValidatableCodec<T>, InheritCodec<T> {
```

## Architecture & Concepts
The ACodecMapCodec is a foundational component of the serialization framework, acting as a **polymorphic dispatcher**. Its primary function is to serialize and deserialize object hierarchies where multiple concrete subtypes inherit from a common base type.

In game development, this pattern is essential for data like entities, blocks, or items. For example, a list of `Block` objects might contain `StoneBlock`, `DirtBlock`, and `GlassBlock` instances. ACodecMapCodec solves the problem of determining which specific codec to use for each object in the data stream.

It operates by inspecting a designated **type identifier field** (e.g., a string field named "Id" or "type") within the serialized data. Based on the value of this field, it dynamically dispatches the serialization or deserialization task to a specialized, registered sub-codec. This mechanism allows the system to correctly reconstruct the specific object subtype from a generic data structure.

This class forms the bridge between abstract data definitions and their concrete, serialized representations, enabling robust and extensible content definition.

## Lifecycle & Ownership
- **Creation:** ACodecMapCodec is an abstract class and cannot be instantiated directly. Concrete implementations are created during the application bootstrap phase, typically within a central codec registry builder. Each implementation is responsible for a specific polymorphic type hierarchy (e.g., an `EntityCodecMap` for all entity types).
- **Scope:** An instance of a subclass persists for the entire application session. Once constructed and populated via the `register` method, it is treated as an immutable component by the rest of the system for all subsequent serialization operations.
- **Destruction:** The object is eligible for garbage collection only upon application shutdown when the master codec registry is dismantled. No explicit cleanup methods are provided or required.

## Internal State & Concurrency
- **State:** The internal state is highly mutable during the initial registration phase. It maintains several concurrent maps to store bidirectional lookups between type identifiers (keys), Java classes, and their corresponding codecs. The primary state includes:
    - A map from an identifier to its codec.
    - A map from a Java class to its identifier.
    - A priority-sorted array of codecs eligible to act as a default fallback.

- **Thread Safety:** This class is **thread-safe**. All internal collections (`idToCodec`, `classToId`, `idToClass`) are implemented using ConcurrentHashMap to allow for safe concurrent read and write operations. The prioritized list of default codecs is managed by an AtomicReference using a copy-on-write strategy, ensuring that modifications (register, remove) do not interfere with concurrent read operations (decode, encode).

    **Warning:** While the component is thread-safe, dynamic registration from multiple threads after the initial bootstrap phase is discouraged. Such a pattern can lead to non-deterministic behavior depending on thread timing, especially concerning which codec is selected as the default. Registration should ideally occur within a single-threaded context during application initialization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(id, aClass, codec) | ACodecMapCodec | O(N) | Registers a subtype. Associates an ID, a Java class, and a specific codec. Complexity is dominated by the potential array copy for the default codec list. |
| remove(aClass) | void | O(N) | Removes a previously registered subtype. |
| decode(bsonValue, extraInfo) | T | O(1) | Deserializes a BSON value into a concrete Java object by dispatching to the appropriate sub-codec based on the identifier field. |
| encode(t, extraInfo) | BsonValue | O(1) | Serializes a Java object into a BSON value, injecting the correct identifier field before returning the final document. |
| getCodecFor(key) | C | O(1) | Retrieves the registered codec for a given identifier. |
| getDefaultCodec() | C | O(1) | Returns the highest-priority codec designated as a fallback, if any. |

## Integration Patterns

### Standard Usage
A concrete implementation must be created. This implementation is then populated with all known subtypes it is expected to handle. It is then used by other codecs that need to serialize a field containing this polymorphic type.

```java
// 1. Define a concrete implementation for a type hierarchy (e.g., Blocks)
public class BlockCodecMap extends ACodecMapCodec<String, Block, Codec<? extends Block>> {
    public BlockCodecMap() {
        super("blockId", Codecs.STRING); // Use "blockId" as the identifier field
    }
}

// 2. During bootstrap, instantiate and register all known subtypes
BlockCodecMap blockCodecs = new BlockCodecMap();
blockCodecs.register("hytale:stone", StoneBlock.class, new StoneBlockCodec());
blockCodecs.register("hytale:dirt", DirtBlock.class, new DirtBlockCodec());

// 3. Use the map codec within another codec
public static final Codec<Chunk> CHUNK_CODEC = BuilderCodec.of(
    Chunk::new,
    "blocks", c -> c.blocks, Codecs.listOf(blockCodecs) // Use the map here
);
```

### Anti-Patterns (Do NOT do this)
- **Late Registration:** Do not register new codecs after the bootstrap phase is complete and the serialization system is actively being used. This can lead to race conditions where data is processed before its corresponding codec is fully registered.
- **Ignoring Exceptions:** Failing to handle the UnknownIdException thrown during decode/encode will lead to crashes. This typically indicates that data is corrupt or a required content pack (mod) is not loaded.
- **Misconfigured Identifier Key:** Using a different identifier key string (e.g., "type" instead of "Id") than what the data source provides will prevent the codec from finding the identifier, causing all decodes to fail or improperly use the default.

## Data Pipeline

### Decoding Flow
> BSON Document -> **ACodecMapCodec**.decode -> Read "key" field -> Lookup Codec in `idToCodec` map -> Delegate to Sub-Codec -> Return hydrated Java Object

### Encoding Flow
> Java Object -> **ACodecMapCodec**.encode -> Get object Class -> Lookup ID in `classToId` map -> Delegate to Sub-Codec for BSON -> Inject "key":ID pair into BSON -> Return final BSON Document


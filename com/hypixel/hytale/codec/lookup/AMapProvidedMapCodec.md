---
description: Architectural reference for AMapProvidedMapCodec
---

# AMapProvidedMapCodec

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Transient

## Definition
```java
// Signature
public abstract class AMapProvidedMapCodec<K, V, P, M extends Map<K, V>> implements Codec<M>, ValidatableCodec<M> {
```

## Architecture & Concepts
AMapProvidedMapCodec is an abstract base class that provides a powerful, generic framework for serializing and deserializing heterogeneous Java Maps. Its primary architectural purpose is to handle data structures where the type of a value is determined by its corresponding key.

This class operates on a principle of **delegated decoding**. It does not contain the logic to process the map's values itself. Instead, it maintains an internal lookup table, the *codecProvider*, which maps a Java key of type K to a provider object of type P. A *mapper* function then transforms this provider object into the specific Codec required to handle the value of type V.

This design is critical for configuration objects, entity component maps, or any system where a single container holds values of many different, but well-defined, types. It decouples the container's serialization logic from the serialization logic of its contents, promoting modularity and extensibility.

The four generic parameters are central to its function:
- **K:** The type of the key in the final Java Map.
- **V:** The common super-type for all values in the Java Map.
- **P:** A "Provider" or "Prototype" type. This is an intermediate object associated with a key, used by the mapper to resolve the correct codec.
- **M:** The concrete Map implementation (e.g., HashMap, EnumMap) that this codec will produce.

## Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses are created and configured during the bootstrap phase of the codec system. They are not intended for direct instantiation by application logic. The critical `codecProvider` map and `mapper` function must be supplied at construction.
- **Scope:** An instance of an AMapProvidedMapCodec is long-lived, typically persisting for the lifetime of the application's codec registry. It is stateless with respect to individual serialization operations and is designed to be reused.
- **Destruction:** The object is managed by the Java garbage collector. No explicit cleanup methods are required. It is reclaimed when the codec registry that holds a reference to it is destroyed.

## Internal State & Concurrency
- **State:** The internal state consists of the `codecProvider` map, the `mapper` function, and the `unmodifiable` flag. This state is established at construction and is treated as **immutable** for the object's lifetime. The class does not store any state related to an in-progress `decode` or `encode` operation.
- **Thread Safety:** This class is **conditionally thread-safe**. Concurrent calls to `encode`, `decode`, and `decodeJson` are safe as long as the underlying Codec instances returned by the `mapper` function are also thread-safe. As the internal configuration is read-only after construction, the AMapProvidedMapCodec itself introduces no race conditions.

## API Surface
The public API is defined by the Codec and ValidatableCodec interfaces. The abstract methods form the contract for concrete implementations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | M | O(N) | Deserializes a BSON document into a Map. N is the number of fields in the document. |
| encode(M, ExtraInfo) | BsonValue | O(N) | Serializes a Map into a BSON document. N is the number of entries in the map. |
| decodeJson(RawJsonReader, ExtraInfo) | M | O(N) | Deserializes a JSON object from a stream into a Map. N is the number of fields. |
| toSchema(SchemaContext) | Schema | O(K) | Generates a JSON Schema definition for the map. K is the number of known keys. |
| handleUnknown(...) | void | O(1) | Protected hook to manage keys found in the source data that are not in the codecProvider. |
| createMap() | M | O(1) | **Abstract.** Factory method for creating the destination Map instance. |
| getIdForKey(K) | String | O(1) | **Abstract.** Defines the mapping from a Java key to its serialized string representation. |
| getKeyForId(String) | K | O(1) | **Abstract.** Defines the mapping from a serialized string key to its Java object representation. |

## Integration Patterns

### Standard Usage
A developer must extend this abstract class and provide concrete implementations for the key mapping and map creation methods. This subclass is then used within a larger codec system.

```java
// A concrete implementation for a map of GameSettings
public class GameSettingsCodec extends AMapProvidedMapCodec<GameSetting, Object, Codec<?>, Map<GameSetting, Object>> {

    // Constructor supplies the lookup map and the identity mapper
    public GameSettingsCodec(Map<GameSetting, Codec<?>> provider) {
        super(provider, Function.identity(), false); // Create a mutable map
    }

    @Override
    public Map<GameSetting, Object> createMap() {
        return new EnumMap<>(GameSetting.class);
    }

    @Override
    protected String getIdForKey(GameSetting key) {
        return key.name().toLowerCase();
    }

    @Override
    protected GameSetting getKeyForId(String id) {
        return GameSetting.valueOf(id.toUpperCase());
    }
    
    // ... other abstract method implementations
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not instantiate concrete subclasses directly in business logic. They should be managed and retrieved from a central codec registry to ensure proper lifecycle and configuration.
- **Mutable Provider Map:** The `codecProvider` map passed to the constructor must not be modified after the codec is in use. Doing so will lead to non-deterministic behavior and severe concurrency issues.
- **Ignoring Unknown Keys:** The default `handleUnknown` behavior silently skips unrecognized data fields. For systems requiring strict validation or forward-compatibility warnings, this method **must** be overridden to log warnings or throw exceptions.
- **Complex Key Mapping:** The `getIdForKey` and `getKeyForId` methods are called for every key during serialization and deserialization. They must be highly performant; avoid file I/O, network calls, or other high-latency operations within them.

## Data Pipeline

The flow of data through this component is a sequence of lookup, delegation, and aggregation.

**Decoding (Deserialization) Flow:**
> BSON/JSON Field -> `getKeyForId(String)` -> Java Key (K) -> `codecProvider.get(K)` -> Provider (P) -> `mapper.apply(P)` -> Value Codec -> `valueCodec.decode()` -> Java Value (V) -> `map.put(K, V)`

**Encoding (Serialization) Flow:**
> Map Entry (K, V) -> `getIdForKey(K)` -> String Key -> `codecProvider.get(K)` -> Provider (P) -> `mapper.apply(P)` -> Value Codec -> `valueCodec.encode(V)` -> BSON Value -> `document.put(String, BsonValue)`


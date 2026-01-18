---
description: Architectural reference for MapProvidedMapCodec
---

# MapProvidedMapCodec

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Transient

## Definition
```java
// Signature
public class MapProvidedMapCodec<V, P> extends AMapProvidedMapCodec<String, V, P, Map<String, V>> {
```

## Architecture & Concepts
The MapProvidedMapCodec is a specialized, high-performance component within the engine's serialization framework. Its primary architectural role is to encode and decode Java Maps with String keys into a binary or structured format.

This class implements a powerful lookup-based strategy for handling heterogeneous maps, where the values may be of different types and require different codecs. It operates on three core principles:

1.  **Provider Map:** It is configured with a `codecProvider` map. During serialization or deserialization, the String key of each map entry is used to look up a corresponding "provider" object of type P from this map.
2.  **Mapper Function:** A `mapper` function is used to transform the retrieved provider object (P) into a concrete Codec instance (`Codec<V>`). This two-step lookup provides a layer of indirection, allowing complex codec selection logic.
3.  **Delegated Instantiation:** Unlike simpler codecs, this class does not instantiate the destination Map itself. Instead, it delegates this responsibility to an injected `Supplier`. This decouples the serialization logic from the concrete Map implementation (e.g., HashMap, LinkedHashMap, ConcurrentHashMap), giving the calling system full control over memory allocation and collection behavior.

It is a concrete implementation of the abstract AMapProvidedMapCodec, specifically tailored for cases where the map key and the lookup ID are identical strings.

### Lifecycle & Ownership
-   **Creation:** An instance of MapProvidedMapCodec is created and configured once during the application's bootstrap phase, typically by a factory or builder responsible for assembling the global Codec Registry. It is not intended for direct instantiation during the game loop.
-   **Scope:** The object is stateless after construction and its lifecycle is bound to the Codec Registry that owns it. It persists for the entire application session.
-   **Destruction:** The object is eligible for garbage collection when the parent Codec Registry is destroyed, for example, during a client disconnect or server shutdown.

## Internal State & Concurrency
-   **State:** Immutable. All dependencies (the provider map, mapper function, and map supplier) are provided at construction and are not modified internally. The class performs no caching and holds no state related to any specific serialization operation.
-   **Thread Safety:** This class is conditionally thread-safe. While the codec's own logic is safe for concurrent use, the overall thread safety of an operation is **entirely dependent on the injected components**.
    -   The `codecProvider` map must be safe for concurrent reads.
    -   The `mapper` function must be pure or thread-safe.
    -   The `supplier` must be thread-safe.
    -   The `Codec<V>` instances returned by the mapper must be thread-safe.

    **WARNING:** Failure to provide thread-safe dependencies will result in race conditions, data corruption, or runtime exceptions in a multi-threaded environment, such as a network processing pipeline.

## API Surface
The primary public contract is defined by the `Codec` interface (e.g., `encode`, `decode`), which is inherited via its parent class. The most significant symbol for this specific implementation is its constructor, which defines its configuration contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MapProvidedMapCodec(...) | Constructor | O(1) | Constructs and configures the codec. Requires a map of codec providers, a function to map providers to codecs, and a supplier for creating the destination map. |

## Integration Patterns

### Standard Usage
This codec is almost never used in isolation. It is designed to be a building block within a larger, composite codec, often for serializing complex configuration objects or entity data.

```java
// Example: Building a codec for a settings map
// where each setting value has its own codec.

// 1. Define the provider map (e.g., setting name -> setting type enum)
Map<String, SettingType> providerMap = new HashMap<>();
providerMap.put("volume", SettingType.FLOAT);
providerMap.put("username", SettingType.STRING);

// 2. Define the mapper (setting type enum -> actual codec)
Function<SettingType, Codec<Object>> mapper = type -> {
    switch (type) {
        case FLOAT: return Codecs.FLOAT;
        case STRING: return Codecs.STRING;
        default: throw new IllegalArgumentException("Unknown setting type");
    }
};

// 3. Define the map supplier
Supplier<Map<String, Object>> mapFactory = HashMap::new;

// 4. Construct the final codec
Codec<Map<String, Object>> settingsCodec = new MapProvidedMapCodec<>(
    providerMap,
    mapper,
    mapFactory
);

// This settingsCodec can now be used to serialize/deserialize the entire settings map.
```

### Anti-Patterns (Do NOT do this)
-   **Usage Outside of Registry Construction:** Do not create new instances of this class within game logic or per-operation. It is a heavy object intended for one-time setup.
-   **Injecting Unsafe Dependencies:** Do not provide a non-thread-safe `Supplier` (like `ArrayList::new`) or a non-thread-safe `codecProvider` map if the resulting codec will be used by multiple network threads. This is a common source of severe concurrency bugs.
-   **Assuming Map Order:** Do not assume that the deserialized map will have a specific iteration order unless you explicitly provide a `Supplier` that guarantees it (e.g., `LinkedHashMap::new`).

## Data Pipeline

The MapProvidedMapCodec acts as a dispatcher in a data serialization or deserialization pipeline.

**Deserialization Flow:**
> Serialized Data Stream -> Parent Codec -> **MapProvidedMapCodec.decode()** -> [For each key-value pair: Look up `Codec<V>` using key] -> `Codec<V>.decode()` -> Populated Map from Supplier

**Serialization Flow:**
> In-Memory Java Map -> **MapProvidedMapCodec.encode()** -> [For each key-value pair: Look up `Codec<V>` using key] -> `Codec<V>.encode()` -> Serialized Data Stream


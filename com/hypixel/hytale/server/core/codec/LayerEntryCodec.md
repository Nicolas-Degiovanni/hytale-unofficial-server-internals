---
description: Architectural reference for LayerEntryCodec
---

# LayerEntryCodec

**Package:** com.hypixel.hytale.server.core.codec
**Type:** Transient

## Definition
```java
// Signature
public class LayerEntryCodec {
```

## Architecture & Concepts
The LayerEntryCodec class is a data transfer object (DTO) that serves as the structured, in-memory representation of a single layer definition. It is not a service or manager, but rather a passive data structure used by higher-level systems, such as world generation or procedural content services.

The key architectural feature is the static final field named CODEC, which is an instance of BuilderCodec. This field declaratively defines the contract for serialization and deserialization, mapping external data keys ("Left", "Right", "UseToolArg") to the internal fields of the Java object. This pattern effectively decouples the data model from the underlying data format (e.g., JSON, NBT), allowing the engine's generic Codec system to hydrate instances of this class without any custom parsing logic.

This class is fundamental to any system that loads layered or stacked definitions from configuration files or network streams.

### Lifecycle & Ownership
-   **Creation:** Instances are created almost exclusively by the Hytale Codec framework during a deserialization process. A higher-level system provides raw data to a generic parser, which then uses the static CODEC field to construct and populate a new LayerEntryCodec instance. Manual instantiation is strongly discouraged.
-   **Scope:** The lifetime of a LayerEntryCodec instance is typically short and bound to the scope of the operation that required it. For example, when a world generation configuration is loaded, a list of these objects may be created, used to generate a single world chunk, and then become eligible for garbage collection.
-   **Destruction:** There is no explicit destruction or cleanup method. Instances are managed by the Java garbage collector and are reclaimed once they are no longer referenced.

## Internal State & Concurrency
-   **State:** Mutable. The internal fields are not declared as final. However, the design intent is for instances to be treated as immutable value objects after they have been populated by the codec system. Modifying an instance after its creation is considered a severe anti-pattern.
-   **Thread Safety:** This class is **not thread-safe** for mutation. If multiple threads attempt to modify the fields of a shared instance, the state will become corrupted. It is safe for concurrent reads, which is the expected use case after deserialization. The static CODEC field itself is thread-safe and can be used by multiple threads to deserialize data concurrently.

## API Surface
The primary public contract is the static CODEC field, which is used by the serialization engine. The instance methods are simple data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDepth() | Integer | O(1) | Returns the configured depth of the layer. Throws NullPointerException if the object was not correctly initialized. |
| getMaterial() | String | O(1) | Returns the material identifier for the layer. Throws NullPointerException if the object was not correctly initialized. |
| isUseToolArg() | boolean | O(1) | Returns the flag indicating a specific tool argument behavior. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. Instead, they will interact with a system that uses it as part of a larger data structure. The codec is used implicitly by the engine's deserialization services.

```java
// Hypothetical example of a manager using the codec
// A developer would call loadConfig, not use the codec directly.

// Inside a hypothetical WorldGenManager:
Codec<List<LayerEntryCodec>> layerListCodec = Codec.listOf(LayerEntryCodec.CODEC);

// The manager uses the list codec to parse a configuration file
List<LayerEntryCodec> layers = HytaleCodecEngine.decode(layerListCodec, configData);

// The manager then uses the hydrated objects
for (LayerEntryCodec layer : layers) {
    world.setBlock(x, y - layer.getDepth(), layer.getMaterial());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new LayerEntryCodec()`. This bypasses the validation and data mapping logic defined in the static CODEC field. All instances should originate from the codec system to ensure data integrity.
-   **Post-Creation Mutation:** Do not modify the state of an instance after it has been deserialized. These objects are intended to be treated as immutable records. Modifying them can lead to unpredictable behavior in systems that have already read their initial state.

## Data Pipeline
LayerEntryCodec acts as a data payload within a larger processing pipeline. It does not perform any logic itself but represents a structured piece of information that has been validated and typed.

> Flow:
> Configuration File (e.g., JSON) -> Engine Deserializer -> **LayerEntryCodec.CODEC** -> **LayerEntryCodec Instance** -> World Generation System -> Final Game State


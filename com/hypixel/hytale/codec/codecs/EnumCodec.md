---
description: Architectural reference for EnumCodec
---

# EnumCodec

**Package:** com.hypixel.hytale.codec.codecs
**Type:** Transient

## Definition
```java
// Signature
public class EnumCodec<T extends Enum<T>> implements Codec<T> {
```

## Architecture & Concepts
The EnumCodec is a specialized component within the Hytale serialization framework responsible for the bidirectional conversion between Java Enum types and their string representations in BSON and JSON formats. It acts as a translator, ensuring that strongly-typed enum constants in the game engine are correctly written to and read from network packets, configuration files, and database documents.

The central design concept is the **EnumStyle**. This internal mechanism allows the codec to gracefully handle two distinct string formats for enums:
1.  **LEGACY:** A deprecated format corresponding to the standard Java enum naming convention, `SCREAMING_SNAKE_CASE`.
2.  **CAMEL_CASE:** The modern, preferred format, `camelCase`, which is more consistent with JSON and other web-centric data standards.

The codec introspects the target enum class upon instantiation to automatically detect the correct style, providing seamless compatibility. Furthermore, it plays a critical role in the engine's data-driven architecture by generating a formal `Schema` for any given enum. This schema generation supports automated validation, UI generation, and developer tooling by explicitly defining the set of all possible values for an enum field.

## Lifecycle & Ownership
-   **Creation:** An EnumCodec is not a global singleton. It is instantiated on-demand by a higher-level authority, typically a `CodecRegistry` or factory, when a codec for a specific enum type is first required. The constructor performs a one-time, moderately expensive reflection operation to cache the enum's constants and their string representations.
-   **Scope:** The lifetime of an EnumCodec instance is tied to the cache of its parent registry. Once created for a specific enum class (e.g., `GameMode.class`), the same instance is reused for all subsequent serialization operations involving that enum for the duration of the application session.
-   **Destruction:** The object is eligible for garbage collection when its parent `CodecRegistry` is destroyed, typically during client or server shutdown.

## Internal State & Concurrency
-   **State:** The EnumCodec is stateful but **effectively immutable** after its initial configuration phase. During construction, it computes and caches the enum constants and their corresponding string keys into final arrays. This pre-computation makes subsequent encoding operations extremely fast. The `documentation` map is the only mutable field, intended to be populated only during the initial setup.

-   **Thread Safety:** The core serialization methods (`encode`, `decode`, `decodeJson`) are **fully thread-safe**. They perform read-only operations on the state that was cached at construction.

    **WARNING:** The configuration method `documentKey` is **not thread-safe**. It mutates the internal documentation map. Calling this method after the codec has been registered and is in active use by multiple threads will lead to race conditions and undefined behavior. All configuration must be completed in a single-threaded context before the codec is used for serialization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EnumCodec(Class) | constructor | O(N) | Creates a codec for an enum, detecting its style. N is the number of enum constants. |
| encode(T, ExtraInfo) | BsonValue | O(1) | Encodes an enum constant to a BsonValue. This is a highly efficient array lookup. |
| decode(BsonValue, ExtraInfo) | T | O(N) | Decodes a BsonValue to an enum constant. Involves a linear scan of cached keys. |
| toSchema(SchemaContext, T) | Schema | O(N) | Generates a data schema describing the enum, its possible values, and documentation. |
| documentKey(T, String) | EnumCodec | O(1) | Adds a documentation string for a specific enum constant. Not thread-safe. |

## Integration Patterns

### Standard Usage
An EnumCodec is typically created and registered within a central codec management system. Direct instantiation is rare in application code.

```java
// In a hypothetical CodecRegistry setup
// 1. Create the codec for a specific enum
EnumCodec<GameMode> gameModeCodec = new EnumCodec<>(GameMode.class);

// 2. (Optional) Provide schema documentation during configuration
gameModeCodec.documentKey(GameMode.CREATIVE, "Enables creative building and flight.")
             .documentKey(GameMode.ADVENTURE, "Restricts block breaking and placement.");

// 3. The codec is now ready for high-throughput serialization
BsonValue encoded = gameModeCodec.encode(GameMode.CREATIVE, ...);
GameMode decoded = gameModeCodec.decode(encoded, ...);
```

### Anti-Patterns (Do NOT do this)
-   **Repetitive Instantiation:** Do not call `new EnumCodec(MyEnum.class)` repeatedly in a loop or per-operation. The constructor uses reflection and is designed to be called only once per enum type. Always cache and reuse codec instances.
-   **Late Configuration:** Do not call `documentKey` after the codec has been passed to a serialization engine or used by multiple threads. This creates a severe race condition.

    ```java
    // BAD: Modifying state while the codec might be in use elsewhere
    EnumCodec<GameMode> codec = registry.getCodec(GameMode.class);
    
    // Another thread could be calling codec.encode() at this exact moment
    codec.documentKey(GameMode.SURVIVAL, "A new description."); 
    ```

## Data Pipeline

The EnumCodec serves as a critical translation point in the data flow between the game engine and external systems.

> **Encoding Flow:**
> Java Enum Instance -> **EnumCodec.encode** -> BsonValue (String) -> BSON Encoder -> Network or Disk

> **Decoding Flow:**
> Network or Disk -> BSON Parser -> BsonValue (String) -> **EnumCodec.decode** -> Java Enum Instance


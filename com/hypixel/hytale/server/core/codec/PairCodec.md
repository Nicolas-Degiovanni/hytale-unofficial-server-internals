---
description: Architectural reference for PairCodec
---

# PairCodec

**Package:** com.hypixel.hytale.server.core.codec
**Type:** Utility

## Definition
```java
// Signature
public class PairCodec {
```

## Architecture & Concepts

The PairCodec class is a static utility that provides concrete serialization and deserialization definitions for common `Pair` types. It is not a codec itself, but rather a namespace and factory for codecs that handle specific, non-generic pair structures, such as a pair of integers or an integer and a string.

This class exists to bridge a gap in the Hytale codec system, which operates on concrete types and does not natively support the serialization of generic containers like `it.unimi.dsi.fastutil.Pair<L, R>`. To overcome this, PairCodec employs a **Serialization Surrogate** pattern. It defines simple, non-generic Data Transfer Objects (DTOs), such as `IntegerPair` and `IntegerStringPair`, which mirror the structure of a generic pair.

Each surrogate DTO contains a public static final `CODEC` field. This field holds a pre-configured `BuilderCodec` instance that declaratively defines the mapping between serialized keys (e.g., "Left", "Right") and the DTO's internal fields. The engine uses these static codec instances to handle the data transformation, while developers can use the provided `toPair` and `fromPair` helper methods to seamlessly convert between the generic `fastutil.Pair` and its serializable surrogate.

## Lifecycle & Ownership

-   **Creation:** The PairCodec class is never instantiated. Its nested surrogate classes (e.g., `IntegerPair`) are instantiated by the codec system during deserialization or by developers calling the `fromPair` static method. The static `CODEC` instances are initialized once upon class loading by the JVM.
-   **Scope:** The static `CODEC` instances are application-scoped and persist for the entire runtime. Instances of the surrogate DTOs are transient and typically have a very short lifecycle, existing only for the duration of a single serialization or deserialization operation.
-   **Destruction:** Surrogate DTO instances are eligible for garbage collection as soon as they are no longer referenced. The static `CODEC` instances are cleaned up when the class loader is unloaded.

## Internal State & Concurrency

-   **State:** The static `CODEC` objects are stateless and immutable after their initial construction. The surrogate DTOs (`IntegerPair`, `IntegerStringPair`) are mutable data holders. Their fields are populated after construction, either by a developer or by the codec system during deserialization.
-   **Thread Safety:** The static `CODEC` instances are inherently thread-safe and can be shared across the application to perform concurrent serialization and deserialization. The surrogate DTO instances are **not thread-safe**. They are simple data containers and must not be shared and modified across threads without external synchronization mechanisms.

## API Surface

The primary API consists of the static `CODEC` fields and the conversion methods within the nested classes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| IntegerPair.CODEC | BuilderCodec | O(1) | Static codec for serializing an `IntegerPair`. |
| IntegerPair.fromPair(Pair) | IntegerPair | O(1) | Static factory to create a surrogate from a generic `Pair`. |
| IntegerPair.toPair() | Pair | O(1) | Converts a surrogate instance back to a generic `Pair`. |
| IntegerStringPair.CODEC | BuilderCodec | O(1) | Static codec for serializing an `IntegerStringPair`. |
| IntegerStringPair.fromPair(Pair) | IntegerStringPair | O(1) | Static factory to create a surrogate from a generic `Pair`. |
| IntegerStringPair.toPair() | Pair | O(1) | Converts a surrogate instance back to a generic `Pair`. |

## Integration Patterns

### Standard Usage

The `CODEC` fields are intended to be composed into other, more complex codec definitions. You do not typically interact with the surrogate DTOs directly unless you are manually converting data.

```java
// Example: Defining a codec for a configuration object that
// contains a pair of integers.

public static final BuilderCodec<MyConfig> CONFIG_CODEC = BuilderCodec.builder(MyConfig.class, MyConfig::new)
    .append(
        new KeyedCodec<>("SpawnRange", PairCodec.IntegerPair.CODEC),
        (config, pair) -> config.spawnRange = pair.toPair(), // Convert back to generic Pair
        (config) -> PairCodec.IntegerPair.fromPair(config.spawnRange) // Convert to surrogate for serialization
    )
    .add()
    .build();
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The top-level `PairCodec` class cannot be instantiated. `new PairCodec()` will fail.
-   **Manual Serialization:** Avoid manually creating an `IntegerPair` instance and then passing it to a serializer. Instead, embed the static `PairCodec.IntegerPair.CODEC` directly into your higher-level object's codec definition as shown in the standard usage pattern.
-   **Assuming Generic Support:** Do not attempt to create a generic `Codec<Pair<T, U>>`. The existence of this utility class is a direct indicator that such a pattern is not supported by the engine.

## Data Pipeline

PairCodec facilitates the translation of in-memory generic `Pair` objects to a serializable format and back.

**Serialization Flow:**
> `Pair<Integer, Integer>` -> `PairCodec.IntegerPair.fromPair()` -> `IntegerPair` DTO -> `IntegerPair.CODEC` -> Serialized Output (e.g., NBT, JSON)

**Deserialization Flow:**
> Serialized Input -> `IntegerPair.CODEC` -> `IntegerPair` DTO -> `dto.toPair()` -> `Pair<Integer, Integer>`


---
description: Architectural reference for AssetKeyValidator
---

# AssetKeyValidator

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient

## Definition
```java
// Signature
public class AssetKeyValidator<K> implements Validator<K> {
```

## Architecture & Concepts
The AssetKeyValidator is a specialized component within the schema and codec framework. It does not perform validation itself; instead, it acts as an adapter that bridges the generic `Validator` interface with the concrete `AssetStore` system.

Its primary function is to enforce referential integrity for asset keys. When the engine parses a data file (such as a JSON model definition) that references another asset (e.g., a texture or sound event), this validator is invoked to confirm that the referenced asset key exists in the corresponding `AssetStore`. This prevents runtime errors caused by missing or misspelled asset identifiers.

A key architectural feature is its use of a `Supplier<AssetStore>`. This decouples the validator's creation from the availability of the `AssetStore` itself. This is critical during engine initialization, where complex dependency graphs and potential circular dependencies can exist. The `AssetStore` is only resolved via the supplier at the moment validation is actually performed.

## Lifecycle & Ownership
- **Creation:** Instantiated by the schema configuration system, typically during the bootstrap phase when data schemas are being built. It is not intended for manual creation by game logic developers. The context that builds the schema is responsible for providing the correct `AssetStore` supplier.
- **Scope:** Short-lived. An instance of AssetKeyValidator is generally scoped to a single validation task, such as parsing one configuration file. It holds no state beyond its initial configuration.
- **Destruction:** The object is lightweight and becomes eligible for garbage collection as soon as the validation process that created it completes. It manages no native resources or persistent connections.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal state consists of a single `final` reference to the `Supplier` provided at construction. The validator itself does not cache data or change its state after creation. All stateful operations are delegated to the underlying `AssetStore`.
- **Thread Safety:** This class is conditionally thread-safe. It contains no internal locks or synchronization primitives. Its safety in a multi-threaded context is entirely dependent on the thread safety of the `AssetStore` instance returned by its `Supplier`. If the underlying `AssetStore` is not safe for concurrent access, then this validator is also not safe.

**WARNING:** Concurrent validation using an AssetKeyValidator that points to a non-thread-safe AssetStore will lead to unpredictable behavior, including data corruption or runtime exceptions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(K, ValidationResults) | void | O(1) | Delegates the validation of the asset key `K` to the underlying AssetStore. Populates the ValidationResults object with errors if the key is not found. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Augments a schema definition with metadata, indicating that the associated field is a reference to a Hytale asset type. Used for tooling and schema introspection. |
| getStore() | AssetStore | O(1) | Resolves and returns the underlying AssetStore via the supplier. Throws NullPointerException if the supplier or its result is null. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this class directly. It is configured declaratively as part of a larger schema definition used by the codec system. The framework handles its instantiation and invocation.

```java
// Hypothetical schema definition
// The framework would translate this into an AssetKeyValidator instance.
SchemaBuilder.field("texture")
    .expectString()
    .withValidator(
        // The framework provides the correct store supplier here
        new AssetKeyValidator<>(assetContext.getStoreSupplier(Texture.class))
    );

// During file loading, the codec invokes the validator automatically.
codec.load(configFile, mySchema); // AssetKeyValidator is used internally here
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually construct an AssetKeyValidator in game logic. The validation system is designed to be configured at startup. Manually creating validators can bypass framework-level caching and lifecycle management.
- **Incorrect Supplier:** Providing a `Supplier` that returns null or an `AssetStore` for the wrong asset type (e.g., a `SoundEvent` store for a `Texture` field) will result in failed or incorrect validation and lead to subtle, hard-to-diagnose bugs.

## Data Pipeline
The AssetKeyValidator sits in the middle of the data deserialization and validation pipeline. It is the component responsible for cross-referencing a value from a data file against a central, authoritative registry (the AssetStore).

> Flow:
> Raw Data File (e.g., JSON) -> Codec Deserializer -> Schema Validation Engine -> **AssetKeyValidator.accept(key)** -> AssetStore.validate(key) -> Populated ValidationResults


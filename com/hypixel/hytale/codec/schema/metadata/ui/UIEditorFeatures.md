---
description: Architectural reference for UIEditorFeatures
---

# UIEditorFeatures

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Transient

## Definition
```java
// Signature
public class UIEditorFeatures implements Metadata {
```

## Architecture & Concepts
The UIEditorFeatures class is a specialized, immutable data container that implements the Metadata interface. Its primary role within the engine's configuration system is to declaratively enable or disable specific features within the Hytale UI editor.

Architecturally, this class embodies the **Strategy Pattern**. It encapsulates a specific configuration "strategy"—in this case, setting UI editor flags—which can be applied to a central configuration object, the Schema. The broader system can collect multiple Metadata implementations, including UIEditorFeatures, and apply them sequentially to build a complete, composite configuration. This decouples the main Schema builder from the specific details of every possible configuration fragment, promoting modularity and extensibility.

This object acts as a type-safe data-transfer object (DTO) for a specific subset of configuration, preventing the use of error-prone primitives like strings or booleans for feature flagging.

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand, typically by a higher-level configuration loader or factory that parses a source asset (e.g., a JSON definition file for a game mode or world). It is not a managed service or singleton.
- **Scope:** Extremely short-lived. The object's lifetime is intended to last only for the duration of the Schema construction or modification process.
- **Destruction:** Once the `modify` method has been invoked on a Schema object, the UIEditorFeatures instance has fulfilled its purpose and is immediately eligible for garbage collection. No long-term references should be held to it.

## Internal State & Concurrency
- **State:** **Immutable**. The internal array of EditorFeature enums is marked as `final` and is populated exclusively through the constructor. The object's state cannot be altered post-instantiation.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, a single instance can be safely read by multiple threads without synchronization. However, the object it modifies, Schema, is almost certainly mutable and not thread-safe. Therefore, any call to the `modify` method must be externally synchronized if the Schema instance is shared across threads.

## API Surface
The public contract is minimal, focusing entirely on its role as a configuration applicator.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UIEditorFeatures(EditorFeature... features) | constructor | O(1) | Constructs a new instance with a specific set of editor features. |
| modify(Schema schema) | void | O(1) | Applies the contained feature flags to the target Schema object. Throws NullPointerException if schema is null. |

## Integration Patterns

### Standard Usage
The intended use is to instantiate the class with the desired features and immediately apply it to a Schema object during a build phase.

```java
// A Schema object is being configured by a builder or factory
Schema targetSchema = new Schema();

// Instantiate UIEditorFeatures with the desired flags
UIEditorFeatures editorMetadata = new UIEditorFeatures(
    UIEditorFeatures.EditorFeature.WEATHER_DAYTIME_BAR,
    UIEditorFeatures.EditorFeature.WEATHER_PREVIEW_LOCAL
);

// Apply this metadata fragment to the main schema
editorMetadata.modify(targetSchema);

// The targetSchema now reflects the enabled UI editor features
```

### Anti-Patterns (Do NOT do this)
- **Holding References:** Do not store instances of UIEditorFeatures in long-lived objects or caches. They are transient configuration fragments, not persistent state.
- **Null Application:** Never call `modify(null)`. The method contract requires a valid, non-null Schema instance. This will cause a runtime crash.
- **Reflective Modification:** Do not use reflection to attempt to change the internal `features` array after construction. This violates the class's immutability guarantee and can lead to unpredictable behavior in the configuration system.

## Data Pipeline
UIEditorFeatures acts as a carrier in the configuration pipeline. It does not process streaming data but rather represents a static piece of configuration that is applied at a specific point in time.

> Flow:
> Configuration Asset (e.g., JSON) -> Asset Deserializer -> **UIEditorFeatures Instance** -> Schema Builder -> Finalized Schema Object


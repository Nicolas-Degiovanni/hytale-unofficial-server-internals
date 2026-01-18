---
description: Architectural reference for UIEditor
---

# UIEditor

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Transient

## Definition
```java
// Signature
public class UIEditor implements Metadata {
```

## Architecture & Concepts
The UIEditor class is a specialized implementation of the Metadata interface, designed to bridge the gap between abstract data schemas and concrete user interface tooling. Its primary function is to annotate a schema with hints about how its data should be presented and manipulated in a visual editor.

This system is built on a polymorphic design centered around the EditorComponent interface. The static CodecMapCodec, a type of discriminated union codec, acts as a registry. It maps string identifiers (e.g., "Timeline", "Dropdown") from a data file to specific EditorComponent implementation classes. This allows content designers to declaratively specify the desired editor widget for a schema property without modifying game code.

When a schema is deserialized, the UIEditor metadata is instantiated with the appropriate EditorComponent. During the final stage of schema processing, the UIEditor's *modify* method is invoked, injecting the specific UI component configuration directly into the Schema object. This enriched Schema can then be consumed by UI generation systems to build rich, context-aware editing tools.

## Lifecycle & Ownership
- **Creation:** UIEditor instances are not created manually. They are instantiated by the Hytale codec system during the deserialization of schema definition files (e.g., JSON, HOCON). The static CodecMapCodec is responsible for creating the correct nested EditorComponent and wrapping it in a UIEditor object.
- **Scope:** The object's lifetime is ephemeral. It exists only during the schema loading and processing pipeline.
- **Destruction:** It is eligible for garbage collection immediately after its *modify* method has been called on the target Schema. It holds no persistent state and is not retained by any long-lived service.

## Internal State & Concurrency
- **State:** The state of a UIEditor instance is immutable after construction. It holds a single final reference to an EditorComponent. The nested EditorComponent classes are mutable during their creation by the BuilderCodec but are treated as immutable data transfer objects thereafter.
- **Thread Safety:** The class is inherently thread-safe for read operations due to its immutability. However, the static *init* method, which populates the shared static CODEC registry, is **not thread-safe**.

    **Warning:** The *init* method performs write operations to a static collection. It must be called exactly once from a single thread during application bootstrap, before any schema deserialization begins. Concurrent calls will lead to a corrupted registry and unpredictable runtime failures.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modify(Schema schema) | void | O(1) | Implements the Metadata contract. Injects the configured EditorComponent into the provided Schema. |
| init() | static void | O(N) | Registers all known EditorComponent types with the master codec registry. N is the number of components. **Critical:** Must be called during engine startup. |

## Integration Patterns

### Standard Usage
This class is primarily used declaratively in data files. The only direct programmatic interaction is the one-time initialization call during the application's startup sequence.

```java
// In your client or server entry point, during the bootstrap phase:
public void onGameInitialize() {
    // ... other initializations
    UIEditor.init();
    // ... proceed to load assets and schemas
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not call `new UIEditor(component)`. The entire purpose of this system is to be driven by data files. Manual creation circumvents the schema definition pipeline and will have no effect.
- **Late Initialization:** Do not call *init* after schema loading has begun. This will result in a `Codec` lookup failure when the deserializer encounters a UI editor component that has not yet been registered. The call must be part of the earliest bootstrap code.
- **Re-Initialization:** Do not call *init* more than once. While it may not cause an immediate crash, it is redundant and can mask bootstrap ordering issues.

## Data Pipeline
The UIEditor functions as a configuration carrier within the broader schema deserialization pipeline.

> Flow:
> Schema Definition File (e.g., JSON) -> Codec System -> **CodecMapCodec** decodes component type -> **BuilderCodec** creates specific EditorComponent -> **UIEditor** is instantiated -> `modify(schema)` is called -> Enriched Schema -> UI Generation System


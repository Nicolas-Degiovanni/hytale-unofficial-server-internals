---
description: Architectural reference for UIRebuildCaches
---

# UIRebuildCaches

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Configuration Object

## Definition
```java
// Signature
public class UIRebuildCaches implements Metadata {
```

## Architecture & Concepts
UIRebuildCaches is a declarative instruction object used within the Hytale UI and rendering configuration system. It does not perform any actions itself. Instead, it serves as a piece of metadata that annotates a configuration property, specifying which client-side graphical caches must be invalidated and rebuilt when that property's value changes.

This class implements the Metadata interface, signaling its role as a modifier within a larger Schema processing pipeline. When a user modifies a setting in the game's UI—for example, changing a global color tint or a texture pack option—the system needs to react by updating relevant graphical assets. UIRebuildCaches provides the explicit link between a configuration property and the rendering caches it affects.

For instance, if a property controlling the color of grass blocks is changed, the MAP_GEOMETRY cache must be rebuilt. This class encapsulates that rule, decoupling the configuration system from the rendering engine. The configuration system simply attaches this metadata; a downstream consumer, such as the AssetManager or RenderingEngine, later queries the modified Schema to determine which rebuild tasks to execute.

### Lifecycle & Ownership
-   **Creation:** Instances are created declaratively and programmatically during the definition of UI configuration schemas. They are typically instantiated alongside a specific configuration property to define the side-effects of modifying that property. They are not created in response to user input at runtime.
-   **Scope:** Transient and short-lived. An instance exists only to be applied to a Schema object via its modify method. It is not retained in memory or persisted across sessions.
-   **Destruction:** Eligible for garbage collection immediately after it has been used to modify a Schema object. Its state is copied into the Schema, at which point the UIRebuildCaches instance itself is no longer needed.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields, including the array of caches and the child properties flag, are marked as final and are set exclusively at construction time. This design guarantees that a configuration rule is atomic and cannot be altered after its definition.

-   **Thread Safety:** **Inherently Thread-Safe**. Due to its immutable nature, a single UIRebuildCaches instance can be safely read by multiple threads without synchronization. However, the modify method it implements is a write operation on the passed-in Schema object. Therefore, the overall operation is only as thread-safe as the Schema object being modified. Callers are responsible for ensuring that the Schema is not mutated concurrently by multiple threads.

## API Surface
The public contract is minimal, consisting only of its constructors and the single method required by the Metadata interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modify(Schema schema) | void | O(1) | Applies the cache rebuild instructions to the provided Schema object. This method mutates the state of the input Schema. |

## Integration Patterns

### Standard Usage
UIRebuildCaches is intended to be used declaratively when defining a schema property. It should be passed as a constructor argument or parameter to a configuration builder, not invoked directly in procedural code.

```java
// Correct Usage: Defining a property and its side-effects.
// When this property is changed, the system will know to rebuild
// the block textures and map geometry caches.
SchemaProperty<Color> grassColor = new SchemaProperty<>(
    "world.grass.color",
    DEFAULT_COLOR,
    new UIRebuildCaches(
        ClientCache.BLOCK_TEXTURES,
        ClientCache.MAP_GEOMETRY
    )
);

// The configuration system will later call modify() internally.
// schema.apply(grassColor);
```

### Anti-Patterns (Do NOT do this)
-   **Procedural Invocation:** Do not create an instance of this class to manually modify a schema. The class is designed to be part of a declarative definition, not an imperative command.

    ```java
    // INCORRECT: Manually creating and applying the metadata.
    Schema mySchema = new Schema();
    UIRebuildCaches caches = new UIRebuildCaches(ClientCache.MODELS);
    caches.modify(mySchema); // Avoid this direct, procedural pattern.
    ```

-   **State Management:** Do not hold references to UIRebuildCaches instances in long-lived objects. They are transient configuration data, not stateful services.

## Data Pipeline
This class functions as a data carrier in the configuration application pipeline. It injects instructions that are acted upon by later stages.

> Flow:
> Configuration Definition -> **UIRebuildCaches** instance created -> User action modifies a property -> `modify(Schema)` is called -> Schema is annotated with rebuild flags -> Rendering Engine consumes flags -> Caches are invalidated and rebuilt.


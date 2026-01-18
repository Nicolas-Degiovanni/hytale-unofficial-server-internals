---
description: Architectural reference for PatternRotationDefinition
---

# PatternRotationDefinition

**Package:** `com.hypixel.hytale.server.core.universe.world.connectedblocks`
**Type:** Configuration Model

## Definition
```java
// Signature
public class PatternRotationDefinition {
```

## Architecture & Concepts

The PatternRotationDefinition class is a declarative configuration model that defines the set of valid rotational and mirroring transformations for a given block pattern. It does not perform transformations itself; rather, it serves as a rule-set that other systems, primarily the connected blocks engine, query to understand a block's behavioral capabilities.

Architecturally, this class is a critical component of the asset serialization pipeline. Its static `CODEC` field, an instance of `BuilderCodec`, allows the Hytale asset loader to deserialize JSON or HOCON configuration files directly into a strongly-typed `PatternRotationDefinition` object. This object is then typically embedded within a larger asset definition, such as a `BlockType`.

The core design is to translate three simple boolean flags—`isCardinallyRotatable`, `isMirrorX`, and `isMirrorZ`—into a comprehensive list of allowed transformations. This abstraction simplifies block definition files while providing the game engine with a precise and iterable set of operations for pattern matching and block state calculation.

### Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the Hytale asset loading system during server or client bootstrap. The `BuilderCodec` uses the private constructor to populate a new instance from a block's asset file. A static `DEFAULT` instance exists as a fallback for block types that do not specify custom rotation rules.
-   **Scope:** The lifetime of an instance is bound to the asset that contains it. It is loaded once when its parent asset (e.g., a `BlockType`) is registered and persists in memory for the entire game session.
-   **Destruction:** The object is marked for garbage collection when the asset registry is cleared or reloaded, such as during a server shutdown or a dynamic resource pack update. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is effectively immutable. Its internal boolean flags are set only once during deserialization via the `BuilderCodec`. The `getRotations` method does not return a stored collection but instead generates a new, lightweight `AbstractList` view on each invocation. This view dynamically calculates its contents based on the immutable configuration flags.
-   **Thread Safety:** The class is inherently thread-safe. Its immutable state guarantees that it can be safely read by multiple threads without synchronization. The static `ROTATIONS` list is populated within a static initializer block, which is guaranteed by the JVM to be a thread-safe, one-time operation. Consequently, this object can be safely used by worker threads involved in chunk generation or world modification without risk of race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRotations() | List<Pair<Rotation, MirrorAxis>> | O(1) | Returns a lightweight, unmodifiable view of all valid transformations. The cost of creating this view is constant and negligible. |

## Integration Patterns

### Standard Usage
This object should be retrieved from a parent asset, such as a `BlockType`. The `getRotations` method is then used to iterate through all possible transformations when evaluating block states or matching world patterns.

```java
// Pseudo-code for a connected block system
BlockType someBlock = AssetRegistry.getBlock("hytale:stone_pillar");
PatternRotationDefinition rotationRules = someBlock.getPatternRotationDefinition();

// Iterate through all valid rotations to find a matching pattern
for (Pair<Rotation, MirrorAxis> transform : rotationRules.getRotations()) {
    if (patternMatchesInWorld(transform)) {
        // Apply transform and set block state
        break;
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PatternRotationDefinition()` in game logic. These objects are data, intended to be loaded from configuration files. Use the static `DEFAULT` instance if a fallback is required.
-   **State Modification:** Do not attempt to modify the state of this object after its creation via reflection. The entire connected blocks system relies on the immutability of these rules.
-   **Caching the Rotation List:** Caching the list returned by `getRotations` is unnecessary. The method is extremely cheap, and creating the list view allocates minimal memory. Caching provides no performance benefit and can complicate the lifecycle of dependent objects.

## Data Pipeline
The flow for this object is driven by the asset loading system. It represents the in-memory materialization of a static configuration file.

> Flow:
> Block Asset File (e.g., `stone_wall.json`) -> Hytale Asset Loader -> **PatternRotationDefinition.CODEC** -> Instantiated **PatternRotationDefinition** -> Embedded in `BlockType` -> Queried by Connected Block Engine


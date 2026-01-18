---
description: Architectural reference for EnvironmentCondition
---

# EnvironmentCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient Data Object

## Definition
```java
// Signature
public class EnvironmentCondition extends Condition {
```

## Architecture & Concepts
The EnvironmentCondition is a specific implementation of the abstract Condition class. It functions as a data-driven predicate used by the server's entity and statistics systems. Its primary role is to determine if an entity's current physical location corresponds to one of a predefined set of game environments, such as a forest, desert, or cave system.

This class acts as a bridge between static game asset data and dynamic world state. An instance of EnvironmentCondition is not typically created programmatically; instead, it is deserialized from asset files (e.g., JSON definitions for mob spawning or item effects) using its static CODEC field. The core evaluation logic, encapsulated in the eval0 method, queries the live game world to retrieve the environment ID at an entity's precise coordinates and compares it against its configured list.

This component is fundamental for creating location-aware game mechanics without hard-coding positional logic.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system during the asset loading phase. The public constructor is not exposed for general use; the static CODEC definition serves as the object's factory.
- **Scope:** An EnvironmentCondition object's lifetime is tied to the parent asset that defines it. It persists in memory as long as the containing asset (e.g., a monster definition) is loaded. It is stateless between evaluations.
- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection once the parent asset is unloaded and no further references to it exist. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** This class is stateful and internally mutable, but designed for a specific initialization pattern.
    - It is initialized with an array of environment names as strings (unknownEnvironments).
    - On the first evaluation, it performs a **lazy initialization** to resolve these string names into a sorted integer array of environment IDs (environments). This integer array is then cached for all subsequent evaluations to improve performance.
    - The `afterDecode` hook in the CODEC definition ensures the cached integer array is nullified upon creation, forcing the lazy initialization on first use.

- **Thread Safety:** **This class is not thread-safe.** The lazy initialization of the environments field within the getEnvironments method is not synchronized. If multiple threads attempt to access an uninitialized instance concurrently, a race condition can occur.

    **WARNING:** All interactions with an EnvironmentCondition instance, particularly the first call to eval0 or getEnvironments, must be confined to a single thread, typically the main server thread responsible for the entity's game tick.

## API Surface
The public API is minimal, focusing entirely on the evaluation contract inherited from the Condition superclass.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | O(log N) | Evaluates the condition. Retrieves entity position, queries the world for the environment ID, and performs a binary search. N is the number of configured environments. |
| getEnvironments() | int[] | O(M log M) first call, O(1) subsequent | **Internal Use.** Lazily resolves and caches environment string names into sorted integer IDs. M is the number of configured environments. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or use. It is automatically managed and invoked by higher-level server systems that process entity logic. The system provides the necessary context, such as the ComponentAccessor and entity Ref, for the evaluation to proceed.

A conceptual example of the system invoking this condition:
```java
// Executed by a higher-level system, e.g., SpawnController or EffectProcessor

// 'condition' is an instance of EnvironmentCondition loaded from an asset
boolean result = condition.eval0(entityComponentAccessor, entityRef, Instant.now());

if (result) {
    // Entity is in one of the specified environments, proceed with logic.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EnvironmentCondition()`. The class is designed to be deserialized from data files. Manual creation will result in an unconfigured and non-functional object.
- **Concurrent Evaluation:** Do not share a single instance across multiple threads and call eval0 concurrently. This can lead to race conditions during the lazy initialization of its internal state.
- **State Modification:** Do not attempt to modify the internal arrays (unknownEnvironments or environments) via reflection after initialization. The object should be treated as immutable after its initial load.

## Data Pipeline
The flow of data for this component begins with asset definition and ends with a boolean game logic decision.

> Flow:
> Game Asset File (JSON) -> Hytale Codec System -> **EnvironmentCondition Instance** -> Entity Evaluation Tick -> eval0() -> World & ChunkStore Query -> Boolean Result


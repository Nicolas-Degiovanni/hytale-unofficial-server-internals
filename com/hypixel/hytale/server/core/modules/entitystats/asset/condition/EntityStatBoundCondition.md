---
description: Architectural reference for EntityStatBoundCondition
---

# EntityStatBoundCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient

## Definition
```java
// Signature
public abstract class EntityStatBoundCondition extends Condition {
```

## Architecture & Concepts
EntityStatBoundCondition is an abstract base class within the server-side Entity Stats module. It serves as a foundational component for creating data-driven conditions that evaluate an entity's specific statistic, such as health, mana, or speed.

Its primary architectural role is to act as a bridge between human-readable configuration and the engine's high-performance runtime systems. Game designers define conditions in asset files using string-based stat names (e.g., "hytale:health"). This class, through its static CODEC, deserializes that configuration into a Java object.

Upon first evaluation, it performs a one-time, lazy lookup to resolve the string name into a highly optimized integer index. All subsequent evaluations use this integer index for direct, fast access into an entity's stat map. This lazy-resolution pattern is critical for balancing configuration flexibility with runtime performance.

Subclasses are expected to implement the final evaluation logic, such as comparing the stat's value against a configured threshold.

### Lifecycle & Ownership
- **Creation:** Instances are not meant to be instantiated programmatically via a constructor. They are created exclusively by the Hytale **Codec** system during the deserialization of game assets, such as AI behavior trees, item effects, or quest objectives.
- **Scope:** The lifecycle of an EntityStatBoundCondition instance is strictly bound to the parent asset that defines it. It is a stateless evaluation object; it holds no data that persists between calls to its evaluation methods.
- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded or goes out of scope. No manual resource management is required.

## Internal State & Concurrency
- **State:** The class contains mutable state, but it is designed to be "write-once". The internal integer field *stat* is lazily initialized from the string field *unknownStat* during the first call to `eval0`. After this initial resolution, the object's state should be considered effectively immutable.

- **Thread Safety:** **This class is not thread-safe.** The lazy initialization logic presents a check-then-act race condition. If multiple threads invoke `eval0` on a newly created instance simultaneously, the string-to-index lookup may execute multiple times, and the write to the *stat* field is not atomic.

    **WARNING:** All evaluation of conditions must be performed within a context that guarantees single-threaded access, such as the main server game loop for a specific entity. Do not share instances of conditions across parallel-processing systems without external locking.

## API Surface
The primary contract is inherited from the parent `Condition` class and specialized here.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | Amortized O(1) | Evaluates the condition against a target entity. The first call has higher latency due to a string-to-index map lookup. Subsequent calls are O(1). Returns false if the entity does not possess the required stat component. |
| eval0(ref, time, value) | abstract boolean | - | Abstract method for subclasses to implement the specific comparison logic against the resolved stat value. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. A game system, such as an AI behavior controller, would retrieve a fully configured `Condition` list from an asset and evaluate it. The internal workings of EntityStatBoundCondition are transparent to the calling system.

```java
// Hypothetical system evaluating a condition loaded from an asset
// The 'condition' instance would be a concrete subclass of EntityStatBoundCondition

// This condition object is deserialized from a game asset by the engine
Condition condition = loadedAsset.getTriggerCondition(); 

// The system evaluates it against a specific entity
boolean shouldTrigger = condition.eval(entityComponentAccessor, entityRef, world.getCurrentTime());

if (shouldTrigger) {
    // Execute game logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create instances of this class or its subclasses using `new`. The object is fundamentally dependent on the Hytale Codec system for proper initialization from asset data.
- **State Mutation:** Do not modify the *unknownStat* or *stat* fields after the object has been created. Doing so will lead to unpredictable behavior.
- **Concurrent Evaluation:** Do not call `eval0` from multiple threads on the same instance. This will cause race conditions during the lazy initialization of the stat index.

## Data Pipeline
The primary flow involves the transformation of declarative asset data into an executable runtime object.

> Flow:
> Game Asset File (e.g., JSON) -> Hytale Codec Deserializer -> **EntityStatBoundCondition** Instance -> Lazy Stat Index Resolution (First `eval0` call) -> Boolean Result


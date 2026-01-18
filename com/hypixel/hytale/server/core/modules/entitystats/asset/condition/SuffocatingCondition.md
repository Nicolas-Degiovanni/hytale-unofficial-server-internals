---
description: Architectural reference for SuffocatingCondition
---

# SuffocatingCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient / Data-Driven Component

## Definition
```java
// Signature
public class SuffocatingCondition extends Condition {
```

## Architecture & Concepts
The SuffocatingCondition is a concrete implementation of the `Condition` contract, designed to act as a server-side predicate for game logic. Its sole purpose is to determine if a given entity is in a state of suffocation. This is achieved by sampling the block and fluid material at the entity's head position within the game world.

This class is a fundamental building block within the Entity Statistics and Status Effect systems. It is not a long-lived service but rather a stateless, reusable piece of logic. The engine's asset pipeline uses the provided `CODEC` to deserialize definitions of this condition from game data files (e.g., JSON or HOCON). This allows game designers to create complex behaviors, such as applying a "drowning" status effect, by declaratively specifying that the SuffocatingCondition must be met.

Its evaluation relies on direct, synchronous lookups into the world's `ChunkStore`, making it a performance-sensitive component that is tightly coupled to the server's world simulation architecture.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the server's asset loading and serialization system via the static `CODEC` field. This occurs when game assets, such as status effects or abilities, are loaded into memory. Direct manual instantiation is heavily discouraged.
- **Scope:** The lifetime of an instance is bound to the lifetime of the containing asset definition. It persists in memory as part of a larger configuration object (e.g., a `StatusEffectAsset`) for the duration of the server session or until assets are reloaded.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when its parent asset is unloaded by the `AssetManager`. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** Immutable. The class itself is stateless. The only configurable field is the `inverse` boolean inherited from the base `Condition` class, which is set at creation and cannot be modified. All evaluations are based entirely on the arguments passed and the current state of the game world.
- **Thread Safety:** Not thread-safe. The `eval0` method performs reads on shared, mutable world data structures like `ChunkStore`. It is critically designed to be executed only from the main server thread that owns the corresponding world instance.

**WARNING:** Invoking the `eval` method from any thread other than the world's primary update thread will result in severe concurrency violations, including `ConcurrentModificationException`, data corruption, or server crashes.

## API Surface
The public contract is defined by the `eval` method in the parent `Condition` class, which internally calls `eval0`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(accessor, ref, time) | boolean | O(1) | Evaluates if the entity is suffocating. Returns true if the entity cannot breathe at its current position. Complexity is constant-time relative to entity count but involves several world data lookups. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be declared within data files and evaluated by higher-level engine systems.

```java
// This class is not used directly in code.
// It is defined in a game asset file (e.g., JSON/HOCON)
// and processed by the engine's status effect module.

// Example conceptual asset definition:
{
  "name": "DrowningEffect",
  "tick": {
    "triggers": [
      {
        "condition": {
          "type": "SuffocatingCondition",
          "inverse": false
        },
        "actions": [
          {
            "type": "DamageAction",
            "amount": 1.0
          }
        ]
      }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SuffocatingCondition()`. The behavior of game mechanics should be driven by asset files, not hard-coded in Java. Rely on the asset pipeline to create and manage these objects.
- **Asynchronous Evaluation:** Never call the `eval` method from a separate thread, a `CompletableFuture`, or any other asynchronous context. All condition evaluations must be synchronized with the main server tick.
- **Stateful Subclassing:** Do not extend this class to add mutable state. The `Condition` system relies on its implementations being stateless and idempotent.

## Data Pipeline
The primary flow for this component is from asset definition to runtime evaluation.

> Flow:
> Game Asset File (JSON/HOCON) -> Server Asset Loader (using `CODEC`) -> In-Memory **SuffocatingCondition** Instance -> Entity Stats System (during server tick) -> `eval()` call -> World & ChunkStore Lookup -> Boolean Result


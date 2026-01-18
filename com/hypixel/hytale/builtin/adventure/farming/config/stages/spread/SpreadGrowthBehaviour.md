---
description: Architectural reference for SpreadGrowthBehaviour
---

# SpreadGrowthBehaviour

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.stages.spread
**Type:** Behavior (Abstract Base)

## Definition
```java
// Signature
public abstract class SpreadGrowthBehaviour {
```

## Architecture & Concepts
The SpreadGrowthBehaviour class is an abstract base component within the server-side farming system. It defines a strategic contract for how a farmable entity, such as a plant, propagates to adjacent locations. This class and its subclasses form a data-driven, polymorphic system that allows game designers to define complex spreading logic in configuration files without modifying core engine code.

Architecturally, it serves as a stateless "strategy" object. Each concrete implementation encapsulates a specific algorithm for spreading. The system is designed around two core responsibilities:

1.  **Positional Validation:** Determining if a target world location is a valid candidate for spreading. This is managed by the base class through the `worldLocationConditions` array.
2.  **Execution:** Performing the actual world modification to create the new entity at a valid location. This logic is deferred to concrete subclasses via the abstract `execute` method.

The heavy reliance on the Hytale `Codec` system, particularly `CodecMapCodec`, is central to its design. This allows the engine to deserialize different spread behaviors from game assets based on a "Type" identifier, enabling a highly configurable and extensible farming framework. The class operates directly on low-level world data structures like `ChunkStore`, indicating its role in performance-critical game logic.

### Lifecycle & Ownership
-   **Creation:** Instances of SpreadGrowthBehaviour are not created directly using the `new` keyword. They are instantiated by the Hytale `Codec` engine during server startup or when game assets are loaded. The static `CODEC` field acts as the deserialization registry for all concrete implementations.
-   **Scope:** These objects are treated as immutable configuration singletons. Once deserialized, they persist for the entire server session, referenced by the farming configurations that use them.
-   **Destruction:** Instances are eligible for garbage collection only when the server shuts down or initiates a full reload of its game configuration assets.

## Internal State & Concurrency
-   **State:** The state of a SpreadGrowthBehaviour instance, consisting of its `worldLocationConditions`, is considered **immutable** after deserialization. The object itself is stateless and holds no information about any specific in-game spread event.
-   **Thread Safety:** The class is inherently thread-safe due to its immutable nature. Public methods do not modify internal fields. **WARNING:** While the behavior object is safe, the `execute` method operates on highly contended, mutable world state (`ChunkStore`, `World`). The calling system, typically the server's main tick loop or a dedicated worker, is responsible for ensuring that modifications to world data are properly synchronized to prevent race conditions. This class provides the logic but not the locking.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | abstract void | Varies | Executes the spread logic. Must be implemented by a subclass. Operates directly on chunk data. |
| validatePosition(world, x, y, z) | protected boolean | O(N) | Tests if a world coordinate is valid for spreading. N is the number of location conditions. |

## Integration Patterns

### Standard Usage
Developers do not call this class directly. Instead, they create a concrete implementation and register it with the `Codec` system. The engine then invokes it based on game configuration.

A typical implementation would define a new spread algorithm and register its own `Codec`.

```java
// 1. Define a concrete behavior
public class MyCustomSpreadBehaviour extends SpreadGrowthBehaviour {
    public static final Codec<MyCustomSpreadBehaviour> CODEC = ... // Codec definition

    @Override
    public void execute(ComponentAccessor<ChunkStore> world, Ref<ChunkStore> sourceChunk, Ref<ChunkStore> targetChunk, int x, int y, int z, float chance) {
        if (world.random().nextFloat() > chance) {
            return; // Failed chance roll
        }
        if (validatePosition(world.get(World.class), x, y, z)) {
            // Place new plant block in the world
            // ... world modification logic ...
        }
    }
}

// 2. Register the codec
SpreadGrowthBehaviour.CODEC.add("my_custom_spread", MyCustomSpreadBehaviour.CODEC);

// 3. Reference it in a JSON/HOCON config file
/*
{
    "growthStage": {
        "spreadBehaviour": {
            "Type": "my_custom_spread",
            "LocationConditions": [ ... ]
        }
    }
}
*/
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new MyCustomSpreadBehaviour()`. The system is entirely dependent on the `Codec` pipeline for initialization and configuration. Direct instantiation bypasses the injection of `worldLocationConditions` and other configured properties.
-   **Stateful Implementations:** A subclass of SpreadGrowthBehaviour must not contain mutable state. These objects are shared and reused across the entire server. Storing instance-specific data (e.g., a countdown timer) within the behavior object will lead to severe concurrency bugs and unpredictable game logic.

## Data Pipeline
The flow of data and control for this component is driven by configuration and the server game loop.

> Flow:
> Game Asset (JSON/HOCON) -> Server Asset Loader -> **Codec Deserialization** -> In-memory **SpreadGrowthBehaviour** instance -> Farming System Tick -> `execute()` call -> ChunkStore Modification -> World State Change


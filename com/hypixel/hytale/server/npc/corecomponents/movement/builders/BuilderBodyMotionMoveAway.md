---
description: Architectural reference for BuilderBodyMotionMoveAway
---

# BuilderBodyMotionMoveAway

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionMoveAway extends BuilderBodyMotionFindWithTarget {
```

## Architecture & Concepts
The BuilderBodyMotionMoveAway class is a key component within the server-side NPC asset definition pipeline. It functions as a deserializer and stateful configurator for the `BodyMotionMoveAway` runtime component, which governs an NPC's "fleeing" or "kiting" behavior.

This class follows the **Builder Pattern**. Its primary responsibility is to translate a JSON configuration block into a valid, in-memory `BodyMotionMoveAway` object. It encapsulates all the tunable parameters that define how an NPC moves away from a target, such as when to slow down, when to stop, and how erratically to behave when cornered.

It inherits from BuilderBodyMotionFindWithTarget, indicating that it shares a foundational set of properties related to target acquisition and pathfinding. The parameters defined within this specific class add the logic for evasive maneuvers.

A critical architectural aspect is its use of `DoubleHolder` and `NumberArrayHolder` fields instead of primitive types. This pattern allows for deferred value resolution. The final configuration values are not resolved until the `build` method is called, enabling dynamic lookups via the `BuilderSupport` context at build time.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderBodyMotionMoveAway is created by the NPC asset loading system when it parses a JSON file and encounters a component definition of this type. It is never intended for manual instantiation during the game loop.
- **Scope:** The lifecycle of this object is extremely short and confined to the asset loading phase. It is a temporary, throwaway object used to configure a single `BodyMotionMoveAway` component.
- **Destruction:** The builder instance becomes eligible for garbage collection immediately after the `build` method is invoked and the resulting `BodyMotionMoveAway` object is attached to its parent NPC. It holds no persistent state beyond this point.

## Internal State & Concurrency
- **State:** This object is highly **mutable**. Its entire purpose is to accumulate state from a JSON source via the `readConfig` method. All of its fields, such as `slowdownDistance` and `erraticDistance`, are populated during this deserialization process.

- **Thread Safety:** This class is **not thread-safe** and must only be used within a single-threaded context, such as the main asset loading thread. The `readConfig` method performs unsynchronized writes to internal fields. Concurrent access will lead to a corrupted or indeterminate state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionMoveAway | O(1) | Constructs the final runtime component. Passes its configured state to the BodyMotionMoveAway constructor. |
| readConfig(JsonElement) | BuilderBodyMotionMoveAway | O(N) | Populates the builder's internal state from a JSON object. N is the number of properties. Returns itself for chaining. |
| getSlowdownDistance(BuilderSupport) | double | O(1) | Resolves and returns the configured slowdown distance via the provided context. |
| getStopDistance(BuilderSupport) | double | O(1) | Resolves and returns the configured stop distance via the provided context. |
| getErraticDistance(BuilderSupport) | double | O(1) | Resolves and returns the distance at which erratic fleeing behavior is triggered. |

## Integration Patterns

### Standard Usage
This class is exclusively intended to be used by the server's asset loading framework. The typical sequence is instantiation, configuration, and building.

```java
// Pseudocode for engine's asset loader
JsonElement moveAwayConfig = parseNpcAssetFile(".../creature.json");
BuilderBodyMotionMoveAway builder = new BuilderBodyMotionMoveAway();

// 1. Configure the builder from the asset data
builder.readConfig(moveAwayConfig);

// 2. Build the final runtime component using the engine's context
BuilderSupport support = engine.getBuilderSupport();
BodyMotionMoveAway runtimeComponent = builder.build(support);

// 3. Attach the component to the NPC
npc.getMovementController().addMotion(runtimeComponent);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not reuse a single builder instance to configure multiple components. The builder is stateful, and this will result in all NPCs sharing the exact same, non-unique movement parameters. Always create a new builder for each component being loaded.
- **Skipping `readConfig`:** Instantiating the class and calling `build` without an intermediate `readConfig` call will produce a component with default, hard-coded values. This bypasses the entire data-driven design of the NPC asset system.
- **Manual Instantiation:** Do not use `new BuilderBodyMotionMoveAway()` in game logic. These builders are part of the asset definition pipeline, not the runtime simulation.

## Data Pipeline
This builder acts as a transformation stage in the data pipeline that converts declarative asset files into live game objects.

> Flow:
> NPC Asset JSON File -> GSON Parser -> `JsonElement` -> **BuilderBodyMotionMoveAway.readConfig()** -> Internal `...Holder` State -> **BuilderBodyMotionMoveAway.build()** -> `BodyMotionMoveAway` Runtime Instance


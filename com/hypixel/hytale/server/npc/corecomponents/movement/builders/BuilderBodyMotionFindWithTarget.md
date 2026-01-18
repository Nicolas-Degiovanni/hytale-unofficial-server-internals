---
description: Architectural reference for BuilderBodyMotionFindWithTarget
---

# BuilderBodyMotionFindWithTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Configuration Builder (Abstract Base)

## Definition
```java
// Signature
public abstract class BuilderBodyMotionFindWithTarget extends BuilderBodyMotionFindBase {
```

## Architecture & Concepts

BuilderBodyMotionFindWithTarget is an abstract base class within the server's NPC asset definition pipeline. It serves as a configuration parser for NPC movement behaviors that are oriented towards a dynamic target, such as chasing a player or following a leader.

This class is not a runtime component. Its sole responsibility is to translate a segment of a JSON asset file into a structured, validated set of parameters. It acts as a strongly-typed intermediary between raw data on disk and the in-memory representation of an NPC's AI logic.

It extends BuilderBodyMotionFindBase, inheriting foundational pathfinding configuration, and adds specialized parameters for managing the relationship with a moving target. These parameters control the conditions under which an NPC will re-evaluate its path, such as when the target has moved a certain distance or left a specific cone of vision. The final configuration values extracted from this builder are used to instantiate and parameterize a concrete BodyMotion component which is then attached to an NPC entity at runtime.

## Lifecycle & Ownership

-   **Creation:** An instance of a concrete subclass is created transiently by the NPC asset loading system when it encounters a corresponding movement component definition within an NPC's JSON behavior file. This process is typically managed by a factory or a registry that maps a JSON "type" string to a specific builder class.
-   **Scope:** The lifecycle of a builder object is extremely short. It exists only for the duration of parsing a single JsonElement. Once the `readConfig` method has been called and the resulting parameters have been retrieved by the asset loader, the builder object has served its purpose.
-   **Destruction:** The object is immediately eligible for garbage collection after the asset loader has finished processing the specific JSON block it was created for. It holds no persistent state and is not retained by any long-lived system.

## Internal State & Concurrency

-   **State:** The internal state is mutable and transient. The class contains several Holder objects (e.g., DoubleHolder, BooleanHolder) which are populated with default values upon instantiation. The `readConfig` method mutates this state by parsing the input JSON and overwriting the default values. This state is not intended to be modified or read after the initial asset loading phase.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated and used within the context of a single-threaded asset loading operation. Concurrent calls to `readConfig` on the same instance will result in a race condition and lead to corrupted, unpredictable configuration state.

## API Surface

The public API is designed for a single, sequential flow: construction, configuration via `readConfig`, and value extraction via getters.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | BuilderBodyMotionFindBase | O(N) | Parses the provided JSON, populating internal state. N is the number of keys in the JSON object. Throws exceptions on malformed data. |
| getMinMoveDistanceWait(support) | double | O(1) | Retrieves the distance a target must move to trigger a path re-computation when the NPC is in a waiting state. |
| getMinMoveDistanceRecompute(support) | double | O(1) | Retrieves the maximum distance a target can move before the NPC's current path is considered stale and must be recomputed. |
| getRecomputeConeAngle(support) | double | O(1) | Retrieves the angle (in radians) defining the cone of vision. If the target moves outside this cone, a re-computation is triggered. |
| isAdjustRangeByHitboxSize(support) | boolean | O(1) | Returns true if pathfinding calculations should account for the physical hitboxes of the NPC and its target. |
| getMinMoveDistanceReproject(support) | double | O(1) | Retrieves the distance a target can move before its position is re-projected onto the navigation mesh. |

## Integration Patterns

### Standard Usage

This class is intended for internal use by the asset loading framework. A developer defining a new NPC movement behavior would create a concrete subclass and register it with the framework. The framework then handles the lifecycle automatically.

```java
// Pseudo-code illustrating the asset loader's internal process

// 1. Asset loader gets a JSON object for a movement component
JsonElement movementJson = npcBehaviorAsset.get("movement");

// 2. A factory creates the correct builder based on a "type" field
//    (This would be a concrete subclass, not the abstract one)
BuilderBodyMotionFindWithTarget builder = builderFactory.create(movementJson);

// 3. The builder parses the JSON into its internal state
builder.readConfig(movementJson);

// 4. The loader extracts the configured values to build a runtime component
RuntimeMovementComponent runtimeComponent = new RuntimeMovementComponent();
runtimeComponent.setRecomputeDistance(builder.getMinMoveDistanceRecompute(support));
runtimeComponent.setConeAngle(builder.getRecomputeConeAngle(support));
// ... and so on for other properties

// 5. The builder is now out of scope and can be garbage collected
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not attempt to reuse a single builder instance to parse multiple JSON configurations. The `readConfig` method is not idempotent and does not clear previous state, which will lead to incorrect configuration blending. Always create a new builder for each distinct JSON object.
-   **Premature Access:** Do not call the getter methods before `readConfig` has been successfully invoked. Doing so will return the hardcoded default values, not the values defined in the asset file, leading to incorrect runtime behavior.
-   **Manual Instantiation:** Application-level code should never instantiate this class directly using `new`. The creation should be delegated to the appropriate asset loading service or factory to ensure proper integration with the engine.

## Data Pipeline

This builder is a key transformation step in the data pipeline that converts declarative JSON assets into executable server-side AI components.

> Flow:
> NPC_Behavior.json -> AssetLoader Service -> JsonElement -> **BuilderBodyMotionFindWithTarget** -> Extracted Configuration Values -> Runtime BodyMotion Component -> NPC Entity AI System


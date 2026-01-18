---
description: Architectural reference for ActionResetSearchRays
---

# ActionResetSearchRays

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class ActionResetSearchRays extends ActionBase {
```

## Architecture & Concepts
ActionResetSearchRays is a concrete implementation of the Command Pattern, designed to function as a leaf node within the Hytale NPC Behavior Tree system. Its sole responsibility is to invalidate cached spatial query results, known as "search rays," for a given NPC.

In the Hytale engine, search rays are used for efficient environmental queries such as line-of-sight checks, terrain analysis, or target validation. The results of these raycasts are often cached by the WorldSupport component to avoid redundant calculations within the same game tick. This action serves as an explicit cache invalidation mechanism.

It can be configured in two modes:
1.  **Global Reset:** If no specific ray IDs are provided, it invalidates all cached search rays for the NPC.
2.  **Targeted Reset:** If an array of IDs is provided, it only invalidates the caches for those specific rays.

This component is configured declaratively within NPC asset files and is executed by the server's Behavior Tree Processor when the execution flow reaches this action.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via code. They are instantiated by the server's asset loading pipeline when an NPC's behavior tree is parsed. A corresponding builder object, BuilderActionResetSearchRays, is populated from the asset file (e.g., a JSON definition) and used to construct this immutable action object.
-   **Scope:** An instance of ActionResetSearchRays persists for the lifetime of the NPC's loaded behavior tree definition. It is effectively a flyweight object, shared and reused by all NPC instances of the same type.
-   **Destruction:** The object is marked for garbage collection when its parent behavior tree definition is unloaded by the server, typically during a full asset reload or server shutdown.

## Internal State & Concurrency
-   **State:** The internal state of this class is **immutable**. The primary field, searchRayIds, is a final array initialized at construction and never modified. The class itself is stateless; all contextual data required for execution (such as the target NPC's Role) is passed as arguments to the execute method.
-   **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable state, a single instance can be safely referenced from multiple threads. However, the NPC Behavior Tree system guarantees that actions for a specific NPC are executed serially on the main server thread. This prevents race conditions on the mutable WorldSupport object that this action manipulates.

## API Surface
The public contract is minimal, consisting only of the `execute` method inherited from ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(N) | Triggers the invalidation of cached search ray positions. N is the number of configured ray IDs. This action always completes successfully and returns true. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is designed to be configured within an NPC's behavior tree asset file and executed automatically by the game engine.

```java
// This class is not meant to be called directly from game logic.
// It is configured in an NPC asset file (e.g., JSON) and run by the engine.

// Conceptual asset definition:
/*
{
  "name": "ClearTargetingRays",
  "type": "ActionResetSearchRays",
  "ids": [ 101, 102, 205 ] // Optional: if omitted, resets all rays
}
*/
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ActionResetSearchRays()`. The object's configuration is tied to the asset pipeline and its builder. Manual creation bypasses this and will likely result in a misconfigured or non-functional component.
-   **Stateful Subclassing:** Avoid extending this class to add mutable state. Behavior tree actions are expected to be stateless, reusable commands.

## Data Pipeline
ActionResetSearchRays does not process a flow of data. Instead, it initiates a state change in a related system. Its execution represents a command flow rather than a data pipeline.

> Flow:
> Behavior Tree Processor → **ActionResetSearchRays.execute()** → Role.getWorldSupport() → WorldSupport.resetCachedSearchRayPosition()


---
description: Architectural reference for ActionRecomputePath
---

# ActionRecomputePath

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ActionRecomputePath extends ActionBase {
```

## Architecture & Concepts
The ActionRecomputePath class is a lightweight, non-blocking command within the server-side NPC AI framework. It serves as a specific instruction within an NPC's behavior tree or state machine.

Its primary architectural role is to decouple the *decision* to find a new path from the computationally expensive *process* of pathfinding. When executed, this action does not perform any pathfinding itself. Instead, it acts as a signaling mechanism, setting a boolean flag on the NPC's active MotionController. This signals to the motion and navigation systems that the current path is stale and a new one should be calculated at the next available opportunity.

This pattern is critical for server performance, ensuring that an AI's decision-making tick is not blocked by a potentially long-running pathfinding query. The actual path computation is deferred to a later phase in the server tick or handled by a separate worker thread, managed entirely by the MotionController.

## Lifecycle & Ownership
- **Creation:** Instances are constructed via the BuilderActionRecomputePath. This typically occurs once during server startup or when world chunks are loaded, as the server parses and builds the static AI behavior trees for different NPC archetypes. It is not intended to be created dynamically during gameplay.
- **Scope:** The object's lifetime is tied to its parent AI behavior definition. It persists as a stateless, reusable component within the AI graph for as long as that AI definition is loaded in memory.
- **Destruction:** The object is garbage collected when the server unloads the corresponding NPC AI definitions, for example, during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** ActionRecomputePath is stateless. It contains no mutable fields and its behavior is determined entirely by the arguments passed to its execute method. Its configuration is immutable after being constructed by its builder.
- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. The execute method is expected to be invoked exclusively from the main server thread during the NPC update phase of the game loop. It modifies the state of the Role and MotionController components, which are also assumed to be confined to the main thread.

## API Surface
The public contract is minimal, consisting only of the inherited execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, info, dt, store) | boolean | O(1) | Sets a flag on the NPC's MotionController, invalidating its current path. Always returns true to signify successful and immediate completion of the action. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be embedded as an action node within an NPC's behavior tree definition file (e.g., HJSON or XML). The AI system automatically calls the execute method when the behavior tree traversal reaches this node.

A conceptual behavior tree might look like this:

```yaml
# Conceptual AI Behavior Definition
- sequence:
  - condition: IsTargetFarFromPath
  - action: ActionRecomputePath
  - action: ActionMoveToTarget
```

### Anti-Patterns (Do NOT do this)
- **Procedural Invocation:** Do not create or call this action from general-purpose game logic. It is tightly coupled to the NPC AI system and its execution context. Using it outside of a behavior tree will bypass the intended deferred execution model and may lead to unpredictable behavior.
- **Expecting Synchronous Results:** Do not assume that a new path is available immediately after this action executes. The pathfinding is asynchronous. Any subsequent logic that depends on the new path must be designed to handle the delay, typically by checking the pathing status in a later tick.

## Data Pipeline
ActionRecomputePath acts as a trigger in a larger data flow. It does not process or transform data itself.

> Flow:
> AI Behavior Tree Tick -> **ActionRecomputePath.execute()** -> MotionController.setForceRecomputePath(true) -> Navigation System Path Query -> MotionController Receives New Path


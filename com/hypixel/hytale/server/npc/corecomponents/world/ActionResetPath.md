---
description: Architectural reference for ActionResetPath
---

# ActionResetPath

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class ActionResetPath extends ActionBase {
```

## Architecture & Concepts
ActionResetPath is a concrete implementation of the **Command Pattern**, designed to function as a leaf node within an NPC's high-level AI structure, such as a Behavior Tree or a Finite State Machine.

Its singular responsibility is to signal the invalidation of an NPC's current navigation path. This class acts as a direct instruction to the NPC's world-interaction layer, forcing the server's pathfinding system to discard the existing path and schedule a recalculation. It does not perform pathfinding itself; it merely triggers the request.

This component serves as a crucial bridge between abstract AI decisions (e.g., "my target has moved behind a new obstacle") and the concrete systems that manage world navigation. It is a fundamental building block for creating dynamic and responsive NPC behaviors that can adapt to a changing environment.

### Lifecycle & Ownership
- **Creation:** Instantiated via its corresponding builder, BuilderActionResetPath. This process is typically managed by a higher-level AI definition loader which parses configuration files (e.g., JSON) that define an NPC's complete behavior tree. It is not intended for direct manual instantiation during gameplay logic.
- **Scope:** The execution of this action is instantaneous. The execute method returns true immediately, indicating the action is completed within a single server tick. The object instance may be held by a behavior tree node for the lifetime of the NPC, but its operational scope is atomic.
- **Destruction:** The object is subject to standard Java garbage collection. It is cleaned up when the parent AI component or the NPC entity itself is destroyed. No manual resource management is required.

## Internal State & Concurrency
- **State:** ActionResetPath is **stateless**. It contains no mutable fields and its behavior is determined entirely by the arguments passed to its execute method, primarily the Role object.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be invoked exclusively from the main server thread that processes NPC updates. All NPC AI logic, including the execution of actions, is confined to a single-threaded, per-world update loop to prevent race conditions and ensure deterministic behavior.

**WARNING:** Calling methods on this object from an asynchronous task or a different thread will lead to severe concurrency issues, including data corruption and server instability.

## API Surface
The public contract is inherited from ActionBase and consists of a single operational method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Triggers a request for a new navigation path. Always returns true, signifying immediate completion. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly from procedural code. It is designed to be embedded within an NPC's declarative AI definition. The system invokes the execute method when the corresponding node in the behavior tree becomes active.

The following example is conceptual, illustrating how the AI runtime would invoke the action.

```java
// Conceptual code within an AI Behavior Tree Node
// This code is part of the engine, not written by a typical user.

ActionResetPath resetAction = configuredAction; // Loaded from definition
boolean isComplete = resetAction.execute(entityRef, npcRole, sensorData, deltaTime, entityStore);

// isComplete will always be true, allowing the behavior tree
// to immediately proceed to the next action.
```

### Anti-Patterns (Do NOT do this)
- **Manual Lifecycle Management:** Do not attempt to create or manage instances of ActionResetPath manually. AI behaviors should be defined in data and loaded by the appropriate server systems.
- **High-Frequency Execution:** Triggering this action repeatedly in a tight loop (e.g., on every tick) is a severe performance anti-pattern. It will cause pathfinding thrashing, where the pathfinding system is constantly recalculating paths for the same NPC, consuming significant CPU resources and preventing the NPC from making any progress.

## Data Pipeline
ActionResetPath initiates a command flow rather than processing a data stream. The flow represents a request that propagates through several server systems.

> Flow:
> AI Behavior Tree Node Activation -> **ActionResetPath.execute()** -> Role.getWorldSupport().requestNewPath() -> Path Invalidated Flag Set -> Pathfinding System schedules recalculation -> New Path (or failure) delivered asynchronously to WorldSupport


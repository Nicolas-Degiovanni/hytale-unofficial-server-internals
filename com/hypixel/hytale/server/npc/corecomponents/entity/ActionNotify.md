---
description: Architectural reference for ActionNotify
---

# ActionNotify

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient Component

## Definition
```java
// Signature
public class ActionNotify extends ActionBase {
```

## Architecture & Concepts
ActionNotify is a concrete implementation of the Command Pattern, operating as a terminal "leaf node" within the server-side NPC Behavior System. Its single, focused responsibility is to facilitate one-way, asynchronous communication between entities. It achieves this by posting a string-based message to a target entity's message inbox, which is managed by the BeaconSupport component.

This class is a critical element of Hytale's decoupled AI design. An NPC executing an ActionNotify does not need to know what the message means, which systems on the target entity will process it, or what the outcome will be. It simply "fires and forgets" the notification. This allows for complex, emergent behaviors where one NPC can signal or trigger state changes in another without a direct, hard-coded dependency.

The target of the notification is resolved dynamically at execution time. It can be sourced from the NPC's short-term memory (a "marked entity" slot) or from its current sensory input (the entity it is currently targeting).

### Lifecycle & Ownership
-   **Creation:** ActionNotify instances are not created directly using `new`. They are instantiated by the server's asset loading pipeline when an NPC's behavior configuration (e.g., a JSON asset) is parsed. A corresponding `BuilderActionNotify` object reads the configuration and, with the help of a `BuilderSupport` context, constructs a fully configured, immutable ActionNotify object.
-   **Scope:** An instance persists for the lifetime of the loaded NPC behavior asset. It is a stateless, reusable component. The same ActionNotify instance is executed every time an NPC's behavior tree logic path reaches this specific action.
-   **Destruction:** The object is eligible for garbage collection when its parent behavior definition is unloaded from memory. This typically occurs during a server-wide asset hot-reload or a full server shutdown.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields (`message`, `expirationTime`, `usedTargetSlot`) are declared `final` and are fully resolved during the asset-loading phase at construction. This class carries no per-execution state, making it inherently reusable and predictable.

-   **Thread Safety:** **Conditionally Safe**. The object itself is immutable and safe to read from any thread. However, the `execute` method is a mutating operation on the wider entity-component system, specifically the target's BeaconSupport component. The server's NPC processing loop is designed to be single-threaded per world or region, which prevents race conditions.

    **Warning:** Manually invoking the `execute` method from multiple threads is an unsupported and dangerous anti-pattern that will lead to state corruption. All interactions must be managed by the NPC's main update tick.

## API Surface
The public contract is minimal, consisting only of the `execute` method inherited from `ActionBase`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Locates a target entity and posts the configured message to its BeaconSupport component. Always returns true, indicating the action completes within a single game tick. |

## Integration Patterns

### Standard Usage
Developers do not typically invoke ActionNotify directly. It is defined within an NPC asset file and executed by the core behavior tree engine. The engine is responsible for supplying the correct context arguments to the `execute` method during the NPC's update tick.

A conceptual example of how the engine might invoke this:
```java
// Hypothetical engine code that runs an NPC's current action
ActionBase currentAction = npc.getBehaviorTree().getCurrentAction();

if (currentAction instanceof ActionNotify) {
    // The engine provides the required context from the current game state
    currentAction.execute(
        npc.getRef(),
        npc.getCurrentRole(),
        npc.getSensorInfo(),
        deltaTime,
        world.getStore()
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never attempt to construct an ActionNotify manually. The required `BuilderActionNotify` and `BuilderSupport` arguments are internal to the asset loading pipeline. All actions must be defined as data in asset files.
-   **Stateful Subclassing:** Do not extend ActionNotify to add state. The behavior system relies on actions being stateless and reusable. For stateful operations, use the NPC's blackboard or other state-management components.
-   **Manual Invocation:** Do not call the `execute` method from custom game logic. It must only be run by the NPC's behavior processor, which guarantees the correct state and threading context.

## Data Pipeline
ActionNotify acts as a bridge, taking a configured message and injecting it into another entity's message queue. The data does not transform within this class; it is merely routed.

> Flow:
> NPC Behavior Tree Execution -> **ActionNotify.execute()** -> Target Entity Lookup (via Role or SensorInfo) -> Component Lookup (for BeaconSupport) -> BeaconSupport.postMessage() -> Target Entity's Internal Message Queue


---
description: Architectural reference for ActionSetStat
---

# ActionSetStat

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient

## Definition
```java
// Signature
public class ActionSetStat extends ActionBase {
```

## Architecture & Concepts
ActionSetStat is a server-side, data-driven command object within the Non-Player Character (NPC) AI framework. As a concrete implementation of ActionBase, its sole responsibility is to modify a numerical statistic on a target entity. This class represents a terminal operation, or a "leaf node," in an NPC's behavior tree or finite state machine.

It is designed to be configured from asset files (e.g., JSON definitions for an NPC's behavior) rather than being instantiated directly in procedural code. The action's parameters—which statistic to change, the value to apply, and whether to add or set the value—are resolved and baked into the object upon creation by the asset loading pipeline.

This component operates directly on the server's Entity Component System (ECS). During its execution, it requests the EntityStatMap component for a given entity and performs the modification, effectively bridging the high-level AI system with the low-level entity data layer.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC asset loading system via its corresponding builder, BuilderActionSetStat. This occurs when an NPC's behavioral definition is parsed and loaded into memory, typically at server startup or when a world zone is initialized. It is not intended for manual creation by developers.
-   **Scope:** The object's lifetime is tied to the loaded NPC definition. It is created once and reused by all instances of that NPC type. The object itself is stateless in the context of a specific NPC instance; its internal fields are configuration data.
-   **Destruction:** The object is eligible for garbage collection when the server unloads the associated NPC asset definitions.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal fields *stat*, *value*, and *add* are final and are set only once in the constructor. This design ensures that a single ActionSetStat instance can be safely reused across multiple NPC entities of the same type without risk of state corruption.

-   **Thread Safety:** The ActionSetStat object itself is thread-safe due to its immutability. However, the execute method modifies shared, mutable state: the EntityStatMap component of a specific entity. The engine's ECS and AI ticking mechanism are responsible for ensuring that modifications to a single entity's components are not subject to race conditions. It is assumed that all actions for a single entity are executed sequentially on a designated thread.

    **Warning:** While the class is safe, the system it operates on requires a strict concurrency model. Calling execute on the same entity from multiple threads simultaneously will lead to data corruption.

## API Surface
The public contract is defined by its parent, ActionBase, and is intended for invocation by the server's AI scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks if the base action can execute and confirms the target entity possesses the specified stat. Fails if the stat does not exist on the entity. |
| execute(...) | boolean | O(1) | Modifies the target entity's stat. Applies the configured value, either additively or by replacement. Asserts that the EntityStatMap component is not null. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in gameplay code. It is executed by higher-level AI systems, such as a Role or Behavior Tree, which are responsible for ticking an NPC's logic. A developer's interaction is limited to defining the action in an asset file.

The conceptual invocation by the engine would look like this:

```java
// Engine-level code (conceptual)
// This code is managed by the NPC's Role or Behavior Tree processor.

// Assume 'currentAction' is an instance of ActionSetStat loaded from assets.
if (currentAction.canExecute(entityRef, role, sensorInfo, deltaTime, worldStore)) {
    currentAction.execute(entityRef, role, sensorInfo, deltaTime, worldStore);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ActionSetStat()`. This bypasses the critical data-driven configuration provided by the BuilderActionSetStat and BuilderSupport system, resulting in an unconfigured and non-functional action.
-   **External State Management:** Do not cache or store the result of canExecute. The state of the entity can change between ticks, and the check must be performed immediately before execution.
-   **Misuse on Non-NPCs:** While technically possible, this action is designed for the NPC AI system. Using it outside this context may bypass expected engine subsystems like event listeners or replication triggers.

## Data Pipeline
ActionSetStat acts as a final step in a data processing flow, translating a high-level AI intent into a low-level data mutation within the ECS.

> Flow:
> AI Scheduler Tick -> Behavior Tree Node -> **ActionSetStat.execute()** -> Store.getComponent() -> EntityStatMap.setStatValue() -> Modified Component State in ECS


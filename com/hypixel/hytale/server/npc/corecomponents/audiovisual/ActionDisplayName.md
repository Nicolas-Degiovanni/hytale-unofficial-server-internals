---
description: Architectural reference for ActionDisplayName
---

# ActionDisplayName

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual
**Type:** Transient Component

## Definition
```java
// Signature
public class ActionDisplayName extends ActionBase {
```

## Architecture & Concepts
ActionDisplayName is a concrete implementation of an *Action* within the server-side NPC Behavior system. It serves a single, highly specific purpose: to change the nameplate text displayed above an NPC in the game world.

As a subclass of ActionBase, it is designed to be a leaf node within a larger behavior graph, such as a Behavior Tree or a Finite State Machine. It represents an instantaneous, fire-and-forget command. When executed, it does not block the behavior graph or wait for a result; it simply submits a request and completes immediately.

The core architectural concept is **nomination**. This action does not directly set the entity's display name. Instead, it *nominates* a new name via the Role and its underlying EntitySupport. This pattern allows a central system to arbitrate between multiple sources that may wish to control the display name simultaneously (e.g., status effects, quest states, or other behaviors).

This class is entirely data-driven, instantiated and configured by the NPC asset loading pipeline from game data files, not created programmatically during gameplay.

## Lifecycle & Ownership
- **Creation:** An ActionDisplayName instance is created by the NPC asset loading system when processing an NPC's definition files. It is constructed using a corresponding BuilderActionDisplayName, which resolves the final string value from asset data. It is not intended for runtime instantiation.
- **Scope:** The object's lifetime is tied to its parent NPC asset definition. As an immutable component, a single instance can be shared and reused across multiple NPC entities of the same type. It persists as long as the NPC definition is loaded in memory.
- **Destruction:** The instance is eligible for garbage collection when the server unloads the associated NPC asset bundle, for example, when a world is shut down or a dynamic zone is unloaded.

## Internal State & Concurrency
- **State:** The class is immutable. Its primary state, the `displayName` string, is a final field initialized at construction. It holds no other runtime state and does not cache data.
- **Thread Safety:** ActionDisplayName is inherently thread-safe due to its immutability. However, the context in which it operates is not. The `execute` method modifies the state of the NPC `Role`. The server's NPC update loop is responsible for ensuring that all behavior actions for a given entity are executed on a single, designated thread to prevent race conditions on the underlying entity components.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Pre-execution check. Returns true if the base conditions are met and the display name was successfully resolved during asset loading. |
| execute(...) | boolean | O(1) | Submits the configured display name to the NPC's EntitySupport for nomination. Always returns true, indicating the action completes in a single tick. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by developers. It is a component used by the NPC Behavior system. The system's executor, such as a Behavior Tree, would invoke it during the server tick update.

```java
// Conceptual example of how the Behavior Tree executor uses the action
// Note: This code does not exist; it is for illustration only.

// Inside an NPC's update tick:
ActionBase currentAction = behaviorTree.getActiveAction();

if (currentAction instanceof ActionDisplayName) {
    if (currentAction.canExecute(ref, role, sensorInfo, dt, store)) {
        currentAction.execute(ref, role, sensorInfo, dt, store);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ActionDisplayName()`. The class relies on the asset pipeline and its corresponding builder to be configured correctly. Manual creation will result in a non-functional component.
- **Runtime Modification:** Do not attempt to alter the `displayName` field via reflection. Instances are shared as part of a static asset definition, and modifying one would affect all NPCs of that type, leading to unpredictable behavior.

## Data Pipeline
The flow of data for this component begins with game assets and ends with a visual change for the player.

> Flow:
> NPC Asset File -> BuilderActionDisplayName -> **ActionDisplayName** -> Behavior Tree Executor -> `role.getEntitySupport().nominateDisplayName()` -> Entity Component System -> Network Packet to Client -> Client-side Nameplate Render Update


---
description: Architectural reference for ActionDropItem
---

# ActionDropItem

**Package:** com.hypixel.hytale.server.npc.corecomponents.items
**Type:** Behavioral Component

## Definition
```java
// Signature
public class ActionDropItem extends ActionWithDelay {
```

## Architecture & Concepts

The ActionDropItem is a stateful, single-purpose component within the server-side NPC Artificial Intelligence framework. It represents a concrete "Action" that can be executed by an NPC, specifically the act of throwing one or more items into the game world.

This class acts as a terminal node within an NPC's Behavior Tree. Its primary responsibility is to bridge the abstract AI decision-making layer with the concrete physics and item management systems. When an NPC's AI decides to drop an item, it invokes this component's execution logic.

Architecturally, it encapsulates three distinct domains:
1.  **AI State Management:** It inherits from ActionWithDelay, providing a built-in cooldown mechanism to prevent action spamming. The `canExecute` method serves as a guard that AI schedulers must respect.
2.  **Ballistic Calculation:** It computes a randomized, physically plausible trajectory for the thrown item using the AimingHelper utility. This involves calculating the required pitch based on distance, gravity, and throw speed.
3.  **World Interaction:** It uses ItemUtils to ultimately spawn the item entity into the world via the core EntityStore, making the action visible and effective.

An instance of ActionDropItem is not generic; it is configured with specific parameters like item type, throw velocity, and target sector, which are loaded from NPC asset definitions.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC asset loading pipeline. An associated `BuilderActionDropItem` reads data from an NPC's definition file (e.g., a JSON or HOCON asset) and constructs an instance of this class. It is never created manually in code.
-   **Scope:** The object's lifetime is tied to its parent NPC's behavior configuration. It persists as a node within the NPC's loaded Behavior Tree. A new instance is created if the NPC's assets are reloaded.
-   **Destruction:** There is no explicit destruction or cleanup method. The object is managed by the Java garbage collector and is reclaimed once the NPC entity is destroyed or its behavior tree is replaced.

## Internal State & Concurrency
-   **State:** This class maintains a hybrid state.
    -   **Immutable Configuration:** Fields defined in the constructor (`item`, `dropList`, `minDistance`, `maxDistance`, `throwSpeed`, etc.) are final and act as the component's static configuration.
    -   **Mutable Runtime State:** The `dropDirection` vector and `pitch` array are mutated during each call to `execute` to calculate a new trajectory. State related to the execution cooldown is inherited from `ActionWithDelay` and is also mutable.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated exclusively by the server's main game loop thread for a specific NPC's update tick. All methods access and mutate component data from the world's `Store`, which is a non-thread-safe operation. Any concurrent access will result in world state corruption, race conditions, and server instability.

## API Surface

The public contract is minimal, intended for use only by the NPC AI scheduler or Behavior Tree runner.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Polls whether the action can be performed. Returns false if the internal cooldown from a previous execution is still active. This must be checked every tick before calling execute. |
| execute(...) | boolean | O(N) | Triggers the item drop logic. Calculates trajectory, spawns item(s) in the world, and activates the internal cooldown. N is the number of items in the configured dropList. |

## Integration Patterns

### Standard Usage

This component is not intended to be invoked directly. It is integrated into an NPC's Behavior Tree and is ticked by the AI scheduler. The conceptual flow within a behavior node would be as follows.

```java
// Conceptual code within an AI Behavior Tree node
// Assume 'actionDropItem' is a member field of this node

// During the NPC's update tick:
if (actionDropItem.canExecute(ref, role, sensorInfo, dt, store)) {
    // The action is not on cooldown, so we can execute it.
    actionDropItem.execute(ref, role, sensorInfo, dt, store);
    
    // The behavior node would typically return a 'SUCCESS' status here.
} else {
    // The action is on cooldown, the node returns a 'FAILURE' or 'RUNNING' status.
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ActionDropItem()`. The constructor's dependencies on builder classes make this impractical, but the principle stands. All actions must be defined in NPC asset files and loaded through the server's asset pipeline.
-   **Bypassing Cooldown:** Do not call `execute` without a preceding, successful `canExecute` check in the same tick. Doing so bypasses the cooldown mechanism managed by the `ActionWithDelay` superclass and can lead to rapid-fire item spawning, breaking game balance and potentially causing performance issues.
-   **State Caching:** Do not cache the result of `canExecute` across ticks. Its return value is time-dependent and will change as the internal cooldown timer progresses. It must be polled on every tick where the action is being considered.

## Data Pipeline

The `execute` method initiates a clear, sequential data flow to translate an AI decision into a world event.

> Flow:
> AI Behavior Tree invokes `execute` -> **ActionDropItem** reads NPC's `TransformComponent` and `HeadRotation` -> A random distance and direction are calculated -> Data is passed to `AimingHelper.computePitch` for ballistic calculation -> The final trajectory vector is computed -> `ItemUtils.throwItem` is called -> A new item entity is created in the `EntityStore`.


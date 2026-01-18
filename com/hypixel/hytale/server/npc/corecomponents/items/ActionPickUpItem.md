---
description: Architectural reference for ActionPickUpItem
---

# ActionPickUpItem

**Package:** com.hypixel.hytale.server.npc.corecomponents.items
**Type:** Transient

## Definition
```java
// Signature
public class ActionPickUpItem extends ActionWithDelay {
```

## Architecture & Concepts
The ActionPickUpItem is a concrete behavior node within the server-side NPC Behavior Tree framework. It encapsulates the logic for an NPC to acquire a dropped item entity from the game world. As a leaf node in the behavior tree, its lifecycle is managed by a higher-level behavior processor which invokes its `canExecute` and `execute` methods during the server's primary game tick.

This action operates in two distinct modes, determined by the `hoover` boolean field:

1.  **Targeted Mode** (hoover is false): The action requires a specific target item entity. This target is typically identified by a preceding `Sensor` node in the behavior tree and passed via the `InfoProvider`. The action's primary responsibility in this mode is to verify that the NPC is within the specified `range` of the target before executing the pickup. This is a direct, intentional behavior.

2.  **Hoover Mode** (hoover is true): The action becomes an ambient, opportunistic behavior. It does not rely on a pre-selected target. Instead, it queries the NPC's `PositionCache` for the nearest dropped item that satisfies optional filtering criteria (`hooverItems`). This mode is less computationally expensive for frequent checks, as the `PositionCache` is optimized for spatial queries.

The action directly manipulates the server's Entity Component System (ECS) via the `Store` and `ComponentAccessor` arguments. It reads `TransformComponent` for positional data and interacts with `ItemComponent` and the NPC's `Inventory` to perform the transfer of the item.

### Lifecycle & Ownership
-   **Creation:** An ActionPickUpItem is never instantiated directly. It is constructed by its corresponding builder, `BuilderActionPickUpItem`, during the server's NPC asset loading phase. The builder pattern resolves all configuration from an asset definition (e.g., a JSON file) into a concrete action instance.
-   **Scope:** The object's lifetime is bound to the `Role` of the NPC that owns it. It is created once when the NPC's behavior set is initialized and persists until the NPC entity is removed from the world.
-   **Destruction:** The object is eligible for garbage collection when its parent `Role` and `NPCEntity` are destroyed. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
-   **State:** The component is stateful. Its configuration (`range`, `storageTarget`, `hoover`, `hooverItems`) is immutable after construction. However, its operational state is mutable, primarily through the delay mechanism inherited from `ActionWithDelay`. This internal timer prevents the action from executing on every tick, turning it into a simple state machine (Ready -> Executing/Delaying -> Ready).
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. All interactions with the `EntityStore` and its components are fundamentally single-threaded. Any attempt to invoke its methods from a worker thread will result in data corruption or a `ConcurrentModificationException`. The heavy use of `assert` on component lookups underscores this design assumption.

## API Surface
The primary contract is defined by the `canExecute` and `execute` overrides, which are invoked by the behavior tree engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerWithSupport(Role) | void | O(1) | Initialization hook. Informs the NPC's Role that this action requires spatial data for dropped items, priming the PositionCache. |
| canExecute(...) | boolean | O(1) | Predicate method. Returns true if the action can be performed, checking the internal delay timer, target validity, and proximity. |
| execute(...) | boolean | O(N) | Performs the pickup. In Hoover mode, complexity is O(N) where N is the number of items in the local PositionCache. Triggers the item transfer and starts the internal delay timer. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is configured within an NPC's asset definition file and managed entirely by the NPC's `Role` and behavior tree processor. The system interacts with it as follows.

```java
// Conceptual example of how the Behavior Tree engine uses the action.
// A developer would NOT write this code.

// During a server tick for a specific NPC...
if (actionPickUpItem.canExecute(ref, role, sensorInfo, dt, store)) {
    actionPickUpItem.execute(ref, role, sensorInfo, dt, store);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ActionPickUpItem()`. The object is complex and requires its state to be injected by the `BuilderActionPickUpItem` during asset loading. Direct instantiation will result in a misconfigured and non-functional object.
-   **External State Management:** Do not call methods like `startDelay` from outside the class. The internal delay state is managed exclusively by the `execute` method to ensure logical consistency.
-   **Off-Thread Execution:** Accessing this action from any thread other than the main server game loop is strictly forbidden and will lead to critical server instability.

## Data Pipeline
The flow of data through this component differs significantly based on its operational mode.

**Targeted Mode (`hoover` = false)**
> Sensor finds an item entity -> `InfoProvider` is populated with the target `Ref` -> Behavior Tree calls `canExecute` -> **ActionPickUpItem** reads NPC and target `TransformComponent` -> Distance is checked against `range` -> `execute` is called -> `ItemComponent.addToItemContainer` is called -> Item is moved to NPC `Inventory`.

**Hoover Mode (`hoover` = true)**
> `PositionCache` is populated with nearby dropped items -> Behavior Tree calls `canExecute` -> **ActionPickUpItem** checks if `PositionCache` is non-empty -> `execute` is called -> **ActionPickUpItem** queries `PositionCache` for the closest item matching `hooverItems` filter -> `ItemComponent.addToItemContainer` is called -> Item is moved to NPC `Inventory`.


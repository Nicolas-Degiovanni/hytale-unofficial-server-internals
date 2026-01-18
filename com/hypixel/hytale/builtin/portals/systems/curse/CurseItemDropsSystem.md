---
description: Architectural reference for CurseItemDropsSystem
---

# CurseItemDropsSystem

**Package:** com.hypixel.hytale.builtin.portals.systems.curse
**Type:** System (Singleton per World)

## Definition
```java
// Signature
public class CurseItemDropsSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The CurseItemDropsSystem is a reactive, event-driven system within the server-side Entity Component System (ECS) framework. Its sole responsibility is to enforce a world-specific game rule: applying a "cursed" status to specific items when they are created as entities within a designated Portal World.

This system acts as a listener for entity creation events. By implementing `getQuery` to filter for entities possessing an ItemComponent, it subscribes to the lifecycle of all item drops. When an item entity is added to the world, the system intercepts the event via `onEntityAdded`. It then consults the world's PortalWorld resource to determine if the item's ID is on a predefined list of "cursed" items for that dimension. If a match is found, it modifies the item's metadata, effectively transforming a standard item into its cursed variant.

This design decouples the game mechanic of "cursing" from the various systems that can create item drops (e.g., block breaking, mob loot, player actions). It centralizes the logic, ensuring the rule is applied consistently across the entire world without requiring any other system to be aware of it.

## Lifecycle & Ownership
- **Creation:** Instances of CurseItemDropsSystem are not created manually. The ECS System Registry discovers and instantiates this system during the world loading process for each active EntityStore.
- **Scope:** The lifecycle of a CurseItemDropsSystem instance is tightly bound to the lifecycle of the world (EntityStore) it services. It persists as long as the world is loaded in memory.
- **Destruction:** The instance is marked for garbage collection when its associated world is unloaded from the server.

## Internal State & Concurrency
- **State:** This system is **stateless**. It contains no member fields and does not cache any data between invocations. All necessary information is retrieved from method parameters (the entity reference) or world-scoped resources (PortalWorld) during execution. This statelessness makes the system highly predictable and robust.
- **Thread Safety:** The Hytale ECS framework guarantees that all system methods for a given world are executed serially on a dedicated world-tick thread. Therefore, this class is not designed to be thread-safe and requires no internal locking mechanisms.

**WARNING:** Calling methods on this system from any thread other than the world's main tick thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is defined by its role as a RefSystem. Direct invocation is not a supported use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(ref, reason, store, cmd) | void | O(1) | Core logic trigger. Executes when an entity with an ItemComponent is added to the world. Modifies the item's metadata if it matches the world's curse list. |
| onEntityRemove(ref, reason, store, cmd) | void | O(1) | No-op. This system does not react to item entity removal. |
| getQuery() | Query | O(1) | Configuration method. Returns a query that selects all entities with an ItemComponent, subscribing the system to their lifecycle events. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system's code directly. Integration occurs at the data level by configuring a PortalWorld resource. The system will be automatically engaged by the engine.

The primary integration point is defining the `cursedItems` set within a world's portal configuration file.

```yaml
# Example: my_hell_dimension.hytale
# This data would be loaded into the PortalWorld resource.
type: portal_world_config
portalType:
  # ... other portal properties
  cursedItems:
    - hytale:shadow_sword
    - hytale:demonic_essence
```
When an item entity for `hytale:shadow_sword` is created in this world, the system will automatically apply the cursed metadata.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new CurseItemDropsSystem()`. The ECS framework is responsible for its creation and lifecycle management. Manually created instances will not be registered with the engine and will have no effect.
- **Manual Invocation:** Do not call `onEntityAdded` directly. This bypasses the ECS scheduler and can cause severe concurrency and state management issues. The engine guarantees the context (Store, CommandBuffer) is valid only during a scheduled system update.

## Data Pipeline
This system processes data in a linear, reactive flow triggered by an in-game event.

> Flow:
> Item Drop Event -> Entity Creation (with ItemComponent) -> ECS Engine dispatches to systems -> **CurseItemDropsSystem.onEntityAdded** -> Reads PortalWorld Resource -> Reads ItemComponent -> Compares Item ID to Cursed List -> Modifies ItemStack Metadata -> Writes back to ItemComponent -> Cursed Item Entity exists in world


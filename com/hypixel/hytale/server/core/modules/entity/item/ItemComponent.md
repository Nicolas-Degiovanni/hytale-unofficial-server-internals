---
description: Architectural reference for ItemComponent
---

# ItemComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** Transient

## Definition
```java
// Signature
public class ItemComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The ItemComponent is a fundamental data component within the server-side Entity-Component-System (ECS) architecture. It is attached to any entity that represents a physical, collectible item lying in the world. This component does not contain logic itself; rather, it holds the state that various game systems query and manipulate to implement item-related behaviors.

It serves as the data-sheet for a dropped item entity, defining what the item is (via an ItemStack), and its current interaction state through a series of timers for pickup and merging. This design decouples the item's data from the systems that act upon it, such as the physics system that moves it, the player interaction system that picks it up, or the optimization system that merges nearby item stacks.

A critical architectural feature is the static `CODEC` field. This enables the component to be serialized and deserialized, which is essential for saving world state to disk and for network replication. The codec's logic also includes provisions for data migration (via `BlockMigrationExtraInfo`), ensuring that items can be updated correctly across different game versions.

## Lifecycle & Ownership
- **Creation:** ItemComponent instances are almost exclusively created via the static factory methods `generateItemDrop` or `generateItemDrops`. These methods are invoked by higher-level game logic when an item needs to be spawned in the world, such as when a player drops an item, a block is destroyed, or a container is broken. These factories correctly assemble a complete item entity `Holder`, bundling the ItemComponent with other essential components like TransformComponent, Velocity, and DespawnComponent. Direct instantiation is a significant anti-pattern.
- **Scope:** The lifecycle of an ItemComponent is strictly bound to the entity it is attached to. It persists as long as the item entity exists within the server's `EntityStore`.
- **Destruction:** The component is destroyed when its parent entity is removed from the `EntityStore`. This occurs under several conditions:
    1.  **Player Pickup:** The item is successfully transferred to a player's inventory via `addToItemContainer`, which then removes the entity.
    2.  **Merging:** The item entity merges with another compatible item entity, and one of them is destroyed.
    3.  **Despawn:** The entity's lifetime, managed by the associated `DespawnComponent`, expires.
    4.  **Manual Removal:** A game system or command explicitly removes the entity for reasons such as cleanup or void damage.

## Internal State & Concurrency
- **State:** The component's state is highly mutable and is expected to be modified frequently by game systems during the server tick.
    - **itemStack:** The core, non-nullable data defining the item's type, quantity, and metadata.
    - **mergeDelay, pickupDelay, pickupThrottle:** These are stateful timers that control gameplay mechanics. They are continuously decremented by a time-aware system to manage when an item can be picked up or merged.
    - **isNetworkOutdated:** A boolean dirty flag. This is a critical optimization pattern. It is set to true when the `itemStack` changes, signaling to the networking layer that this component's state needs to be synchronized with clients. The flag is consumed (read and reset) by the network system.
    - **pickupRange:** A lazily-initialized cache for the item's pickup radius, derived from game configuration files.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. All reads and writes to an ItemComponent instance must be performed on the main server game thread. Unsynchronized access from other threads will lead to race conditions, data corruption, and severe client-server desynchronization. This is a deliberate design choice common in high-performance ECS architectures that rely on single-threaded access to component data during the game tick.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setItemStack(ItemStack) | void | O(1) | Updates the item stack and crucially marks the component as network-outdated. |
| pollPickupDelay(float dt) | boolean | O(1) | Decrements the pickup delay timer. Returns true if the timer has elapsed. |
| pollMergeDelay(float dt) | boolean | O(1) | Decrements the merge delay timer. Returns true if the timer has elapsed. |
| canPickUp() | boolean | O(1) | Checks if the pickup delay has elapsed, indicating the item is available for pickup. |
| consumeNetworkOutdated() | boolean | O(1) | Atomically retrieves and resets the network dirty flag. Used exclusively by the networking layer. |
| generateItemDrop(...) | static Holder | O(1) | Factory method to create a new, fully-formed item entity at a specific position. |
| addToItemContainer(...) | static ItemStack | O(N) | Attempts to add the item entity to an inventory. Handles partial transfers and entity destruction. |

## Integration Patterns

### Standard Usage
Developers should never instantiate or manage an ItemComponent directly. The primary interaction is through the static factory and utility methods, which orchestrate the creation and destruction of the entire item entity.

```java
// Spawning a new item drop in the world
Vector3d spawnPosition = new Vector3d(10, 64, 10);
Vector3f rotation = Vector3f.ZERO;
List<ItemStack> drops = List.of(new ItemStack("hytale:stone", 16));

// The accessor provides context like the world and entity registries
Holder<EntityStore>[] droppedEntities = ItemComponent.generateItemDrops(accessor, drops, spawnPosition, rotation);

// The returned holders must be added to the world's entity store to become active
for (Holder<EntityStore> entity : droppedEntities) {
    accessor.getExternalData().addEntity(entity);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ItemComponent()`. This bypasses the factory methods that correctly assemble the entity with all its required companion components (e.g., TransformComponent, DespawnComponent, PhysicsValues). An entity with only an ItemComponent will not function correctly.
- **Asynchronous Modification:** Do not modify an ItemComponent from a separate thread. All game logic, including decrementing timers or changing the ItemStack, must occur on the main server tick thread to prevent data corruption.
- **Bypassing Setters:** Directly modifying the internal ItemStack object returned by `getItemStack` is dangerous. This will not trigger the `isNetworkOutdated` flag, causing the client's view of the item to become desynchronized from the server. Always use `setItemStack` for any modifications.

## Data Pipeline
The ItemComponent is a central node in several key data flows for in-world items.

> **Flow 1: Item Spawning**
> Game Event (e.g., Block Destruction) -> `ItemComponent.generateItemDrop` -> New `Holder` created with `ItemComponent` and others -> `EntityStore` accepts the new entity -> Physics System reads `Velocity` and updates `TransformComponent` -> Network System detects new entity, serializes `ItemComponent` via its `CODEC`, and sends "spawn entity" packet to clients.

> **Flow 2: Player Pickup**
> Player Proximity System detects an entity with `ItemComponent` -> System checks `itemComponent.canPickUp()` -> If true, calls `ItemComponent.addToItemContainer` with player's `ItemContainer` -> Logic attempts to add `ItemStack` to inventory -> **Path A:** Item is fully absorbed -> `store.removeEntity` is called -> Network System sends "destroy entity" packet. **Path B:** Item is partially absorbed -> `itemComponent.setItemStack` is called with the remainder -> `isNetworkOutdated` flag is set -> Network System detects dirty component and sends "update entity component" packet.


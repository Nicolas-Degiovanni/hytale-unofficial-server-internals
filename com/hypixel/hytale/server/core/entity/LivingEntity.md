---
description: Architectural reference for LivingEntity
---

# LivingEntity

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Domain Entity (Abstract)

## Definition
```java
// Signature
public abstract class LivingEntity extends Entity {
```

## Architecture & Concepts
The LivingEntity class is the abstract foundation for all dynamic, interactive entities within the game world that possess health, stats, and the ability to hold items. It serves as the base for Players, NPCs, and Monsters, extending the core `Entity` class to introduce sophisticated gameplay mechanics.

Architecturally, LivingEntity acts as a central integration point for three critical server-side systems:

1.  **Inventory Management:** It owns and manages an `Inventory` object, which is the canonical source of truth for all items held, worn, or equipped by the entity.
2.  **Stat Modification:** It contains a `StatModifiersManager`, which calculates the entity's final gameplay statistics (e.g., damage, defense, speed) by aggregating base stats and modifiers from equipment and status effects.
3.  **Stateful Mechanics:** It directly implements or provides hooks for state-dependent game logic, such as fall damage accumulation, breathing rules, and item durability.

The class is designed to be subclassed, with concrete implementations like `Player` providing specific inventory layouts and behaviors. Its primary responsibility is to orchestrate the interactions between its internal components, for example, by listening for armor changes in its `Inventory` to trigger a recalculation in its `StatModifiersManager`.

### Lifecycle & Ownership
-   **Creation:** As an abstract class, LivingEntity is never instantiated directly. Concrete subclasses are instantiated by the `EntityStore` when a new entity is spawned in the world or deserialized from storage. The static `CODEC` field is the primary mechanism for hydrating an entity and its inventory from persistent data. Upon creation, a default inventory is immediately generated and assigned via the abstract `createDefaultInventory` method.
-   **Scope:** An instance of LivingEntity persists for the entire duration of that entity's existence within a loaded world chunk. Its state is saved to the `EntityStore` when the chunk is unloaded and restored upon reload.
-   **Destruction:** An entity is marked for destruction when it is killed or administratively removed. The `EntityStore` is responsible for removing it from the world tick and allowing the Java garbage collector to reclaim its memory. The `setInventory` method includes logic to unregister event listeners from old inventories, preventing memory leaks during the entity's lifetime.

## Internal State & Concurrency
-   **State:** LivingEntity is a highly mutable, stateful object. Its core state includes its `Inventory`, `currentFallDistance`, and the cached calculations within its `StatModifiersManager`. It also maintains a network-synchronization dirty flag, `isEquipmentNetworkOutdated`, to optimize network traffic.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be accessed and manipulated exclusively by the main server game loop thread. Any attempt to modify its state, such as changing its inventory or position from an asynchronous task or network I/O thread, will result in state corruption, race conditions, and server instability. All interactions must be scheduled as tasks to be executed on the main game thread.

## API Surface
The public API of LivingEntity is designed for server-side game logic systems to interact with the entity's core mechanics.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setInventory(Inventory inventory) | Inventory | O(1) | Replaces the entity's inventory. **Warning:** This is a destructive operation that unregisters listeners on the old inventory. |
| moveTo(ref, x, y, z, accessor) | void | O(1) | Overrides Entity.moveTo to add fall damage calculation based on vertical displacement. |
| decreaseItemStackDurability(...) | ItemStackSlotTransaction | O(1) | Applies durability damage to a specified item. Triggers stat recalculation if an item breaks. |
| invalidateEquipmentNetwork() | void | O(1) | Sets a dirty flag, indicating that the entity's equipment has changed and needs to be synchronized with clients. |
| consumeEquipmentNetworkOutdated() | boolean | O(1) | Atomically reads and resets the equipment network dirty flag. Intended for use by the networking layer. |
| getStatModifiersManager() | StatModifiersManager | O(1) | Returns a reference to the manager for entity stats. Do not cache this reference across ticks. |
| canBreathe(...) | boolean | O(1) | Determines if the entity can survive in the material at its current head position. |

## Integration Patterns

### Standard Usage
Game systems (e.g., Combat, Physics) should retrieve a LivingEntity instance via a `ComponentAccessor` during the server tick and operate on it directly. Logic should be contained within a system that processes a query of entities.

```java
// Example: A system that applies fall damage at the end of a tick
Ref<EntityStore> entityRef = ...; // Reference to an entity
ComponentAccessor<EntityStore> accessor = ...; // Accessor for the current tick

// Retrieve the LivingEntity component; may be null if the entity is not "living"
LivingEntity livingEntity = accessor.getComponent(entityRef, LivingEntity.getComponentType());

if (livingEntity != null && livingEntity.getCurrentFallDistance() > 3.0) {
    // A hypothetical damage system would handle the logic
    // DamageSystem.applyFallDamage(livingEntity, livingEntity.getCurrentFallDistance());
    livingEntity.setCurrentFallDistance(0.0);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new Player()` or `new Monster()` to create an entity. All entity creation must go through the `World` or `EntityStore` APIs to ensure they are properly registered and tracked.
-   **Asynchronous State Modification:** Never call methods like `setInventory` or `moveTo` from a separate thread. This will bypass engine safeguards and corrupt game state.
-   **Caching References:** Do not cache the return value of `getInventory()` across ticks. The underlying inventory object can be replaced at any time, leading to stale references and unpredictable behavior. Always retrieve it from the entity instance when needed.

## Data Pipeline
LivingEntity often acts as the orchestrator in a chain of events. A common example is armor taking damage, which can affect entity stats.

> Flow:
> Combat System Hit -> **LivingEntity.decreaseItemStackDurability()** -> Inventory.replaceItemStackInSlot() -> Inventory Change Event Fired -> **LivingEntity Event Listener** -> StatModifiersManager.setRecalculate(true) -> Stat System (Next Tick) reads updated stats
---


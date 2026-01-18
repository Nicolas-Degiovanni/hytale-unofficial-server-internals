---
description: Architectural reference for the Flock component, which manages group AI state.
---

# Flock

**Package:** com.hypixel.hytale.server.flock
**Type:** Component (Data)

## Definition
```java
// Signature
public class Flock implements Component<EntityStore> {
```

## Architecture & Concepts
The Flock component is a server-side data container that represents the collective state of a group of entities, or a "flock". It is a fundamental building block of the Entity-Component-System (ECS) architecture used for Non-Player Character (NPC) group behavior.

This component does not contain any logic itself. Instead, it serves as a centralized state manager that is attached to a single "leader" entity. Various AI and combat systems read from and write to this component to coordinate the actions of the entire flock.

Its most critical architectural feature is the use of a **double-buffering** pattern for tracking combat statistics, specifically through the `currentDamageData` and `nextDamageData` fields. This pattern decouples state mutation from state reading within a single game tick. Systems that generate events (like a kill) write to the *next* buffer, while systems that make decisions (like AI behaviors) read from the *current* buffer. This ensures that the AI operates on a consistent snapshot of the world state for the duration of its update cycle, preventing race conditions and temporal inconsistencies. The `swapDamageDataBuffers` method acts as the atomic commit operation at the boundary of a game tick.

## Lifecycle & Ownership
- **Creation:** A Flock component is instantiated and attached to an entity when a new flock is formed in the game world. This is typically driven by a `FlockAsset` definition, which acts as a template, and is managed by higher-level spawner or world-generation systems. It is never created in isolation; it is always added to an entity via a `ComponentAccessor`.

- **Scope:** The component's lifetime is strictly bound to the entity it is attached to. It persists as long as the leader entity exists and is loaded in the world.

- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the `EntityStore`. This can happen if the flock is dissolved, the leader is killed, or the world chunk containing the entity is unloaded. The `FlockRemovedStatus` enum tracks the reason for its removal.

## Internal State & Concurrency
- **State:** The Flock component is highly **mutable**. Its primary function is to aggregate and cache the dynamic state of the flock, including membership, leadership, and real-time combat data. The `PersistentFlockData` object holds long-term state, while the `DamageData` objects hold transient, per-tick information.

- **Thread Safety:** This component is **not thread-safe** and must only be accessed from the main server game loop thread. The ECS framework guarantees single-threaded access during the entity update phase. Any attempt to modify this component from an asynchronous task or a different thread without explicit, external locking mechanisms will result in data corruption, buffer overflows, and severe, unpredictable AI behavior. The integrity of the double-buffering mechanism relies on the synchronous, ordered execution of the game tick.

## API Surface
The public API is designed for interaction by various game systems within the main loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for the Flock component, used for ECS lookups. |
| getDamageData() | DamageData | O(1) | Returns the **current** tick's aggregated damage data. Intended for read-only access by AI systems. |
| onTargetKilled(...) | void | O(1) | Records a kill event. This method writes to the **next** damage buffer, not the current one. |
| swapDamageDataBuffers() | void | O(1) | Atomically swaps the current and next damage buffers. This is a critical state transition that must be called exactly once per tick by the managing system. |
| getFlockData() | PersistentFlockData | O(1) | Provides access to the core, persistent state of the flock, such as its member list and configuration. |
| setRemovedStatus(...) | void | O(1) | Sets the status flag indicating why the flock has been removed. |

## Integration Patterns

### Standard Usage
A `FlockSystem` or a similar AI-coordinating system is the primary consumer. The pattern is to read from the current buffer, allow other systems to write to the next buffer, and then swap them at the end of the tick.

```java
// In an AI System (e.g., FlockBehaviorSystem)
// This system READS the stable, current state to make decisions.
Flock flock = componentAccessor.getComponent(leaderEntity, Flock.getComponentType());
if (flock != null) {
    DamageData currentDamage = flock.getDamageData();
    // ... make decisions based on currentDamage ...
}

// In a Combat System
// This system WRITES to the next buffer when an event occurs.
// This can happen at any point during the tick.
public void handleEntityDeath(Ref<EntityStore> victim, Flock killerFlock) {
    killerFlock.onTargetKilled(componentAccessor, victim);
}

// In a master update loop (e.g., PostUpdateSystem)
// This system COMMITS the changes for the next tick.
public void postUpdate() {
    for (Flock flock : allActiveFlocks) {
        flock.swapDamageDataBuffers();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new Flock()`. Components must be created and managed by the `EntityStore` and `ComponentAccessor` to ensure they are correctly registered within the ECS framework.

- **Incorrect Buffer Access:** Do not write to the buffer returned by `getDamageData()`. This buffer is considered immutable for the duration of the tick. All writes must go through dedicated methods like `onTargetKilled` which target the next buffer.

- **Mishandling Buffer Swaps:** Calling `swapDamageDataBuffers` more than once per tick or from the wrong system will break the state snapshot, leading to lost data or double-counted events. This method should only be called by a single, authoritative system at a designated point in the game loop, typically at the very end.

## Data Pipeline
The flow of combat data through the component is strictly managed by the double-buffering mechanism to ensure tick-by-tick consistency.

> Flow:
> 1. **Combat Event** (e.g., an entity is killed)
> 2. -> **Combat System** detects the event
> 3. -> Calls `flock.onTargetKilled()`
> 4. -> Data is written to the **`nextDamageData`** buffer inside the **Flock** component
> 5. -> **AI System** (during the same tick) calls `flock.getDamageData()`
> 6. -> Reads stable data from the **`currentDamageData`** buffer to make decisions
> 7. -> **End of Tick**
> 8. -> **Master Flock System** calls `flock.swapDamageDataBuffers()` on all flocks
> 9. -> The internal pointers for `currentDamageData` and `nextDamageData` are swapped, and the new `nextDamageData` buffer is reset. The data from step 4 is now visible for the next tick's AI update.


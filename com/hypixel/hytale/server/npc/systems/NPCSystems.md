---
description: Architectural reference for NPCSystems
---

# NPCSystems

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Utility / System Container

## Definition
```java
// Signature
public class NPCSystems {
    // Contains multiple static inner classes that are individual ECS Systems
}
```

## Architecture & Concepts

NPCSystems is not a singular, instantiable class but a static container for a suite of highly specialized systems within the server's Entity Component System (ECS) framework. Its sole purpose is to group all core logic related to the lifecycle and event handling of Non-Player Character (NPC) entities.

Each inner class within NPCSystems represents a distinct piece of logic that reacts to changes in the game state. These are not traditional objects to be called directly; instead, they are registered with the ECS scheduler during server initialization (typically by the NPCPlugin). The scheduler then invokes them automatically when specific conditions are met, such as an entity gaining or losing a component, or a global event being dispatched.

This class and its subsystems are central to the server's NPC behavior, acting as the primary mechanism for initialization, state transition, and cleanup. They form the bridge between raw data components (like NPCEntity, TransformComponent) and complex game logic (like applying spawn effects, setting up AI blackboards, or handling death).

### Lifecycle & Ownership
- **Creation:** The container class NPCSystems is never instantiated. The individual inner system classes (e.g., AddedSystem, OnDeathSystem) are instantiated by the ECS framework during plugin loading and system registration. This process is managed by the server's core module loader.
- **Scope:** Once registered, these systems persist for the entire lifetime of the server's ECS world. They are stateless singletons within the context of the ECS scheduler.
- **Destruction:** The systems are destroyed and cleaned up only when the server's ECS world is shut down.

## Internal State & Concurrency
- **State:** The NPCSystems container is stateless. The inner system classes are also designed to be **completely stateless**. They hold no per-entity data. All state they operate on is passed into their handler methods via the Store (for reading current state) and CommandBuffer (for queueing state changes).
- **Thread Safety:** These systems are inherently thread-safe due to their stateless design. Concurrency and data consistency are not managed by the systems themselves but are guaranteed by the parent ECS scheduler. The scheduler executes systems in a deterministic order and uses a CommandBuffer pattern to defer all entity mutations. This prevents race conditions and ensures that all modifications are applied atomically at the end of a tick.

**WARNING:** Direct mutation of component data from within a system is a critical anti-pattern. All entity modifications **must** be queued on the provided CommandBuffer.

## API Surface

The public contract of NPCSystems is defined by the collection of systems it contains. Direct interaction is not intended. A developer interacts with these systems implicitly by manipulating entity components.

| System Class | Base Type | Trigger | Purpose |
| :--- | :--- | :--- | :--- |
| AddSpawnEntityEffectSystem | RefSystem | `NPCEntity` + `EffectControllerComponent` added | Applies a visual spawn effect to the NPC, configured via its associated BalanceAsset. |
| AddedFromExternalSystem | RefSystem | `NPCEntity` + `FromWorldGen` or `FromPrefab` added | Performs one-time initialization for NPCs spawned by world generation or prefabs, such as setting the initial leash point. |
| AddedFromWorldGenSystem | HolderSystem | `NPCEntity` + `FromWorldGen` added | Assigns a persistent `WorldGenId` component to NPCs originating from world generation, allowing them to be tracked. |
| AddedSystem | RefSystem | `NPCEntity` component added | Core NPC initialization logic. Sets up AI blackboard views, adds required components like `PositionDataComponent`, and applies a temporary spawn lock. |
| KillFeedDecedentEventSystem | EntityEventSystem | `KillFeedEvent.DecedentMessage` on an NPC | Suppresses kill feed messages where an NPC is the victim. NPCs should not appear as decedents in the global feed. |
| KillFeedKillerEventSystem | EntityEventSystem | `KillFeedEvent.KillerMessage` on an NPC | Formats the kill feed message when an NPC kills a player, using the NPC's display name or role name. |
| LegacyWorldGenId | HolderSystem | `NPCEntity` added without a `WorldGenId` | A deprecated compatibility system to convert an old `legacyWorldgenId` field into the modern `WorldGenId` component. |
| ModelChangeSystem | RefChangeSystem | `ModelComponent` added or changed on an NPC | Reacts to model changes (e.g., transformations) to update the NPC's motion controllers and bounding boxes. |
| OnDeathSystem | DeathSystems.OnDeathSystem | `DeathComponent` added to an NPC | Triggers death logic. If the NPC's role specifies a death animation, this system adds a `DeferredCorpseRemoval` component to prevent immediate despawning. |
| OnTeleportSystem | RefChangeSystem | `Teleport` component added to an NPC | Notifies the NPC's Role that a teleport has occurred, allowing AI and other logic to reset or adapt to the new location. |
| PrefabPlaceEntityEventSystem | WorldEventSystem | `PrefabPlaceEntityEvent` is fired | Remaps flock IDs for NPCs that are part of a placed prefab, ensuring flock behaviors work correctly across prefab instances. |

## Integration Patterns

### Standard Usage

Developers should never instantiate or call these systems directly. Interaction is always data-driven by adding, removing, or modifying components on an entity. The ECS scheduler handles the rest.

```java
// Correct Usage: Modify an entity's components to trigger a system.
// This example will implicitly trigger NPCSystems.OnDeathSystem.

// Assume 'npcRef' is a valid reference to an entity with an NPCEntity component.
// Assume 'commandBuffer' is provided by the current system's execution context.

// To kill an NPC, you add a DeathComponent.
DeathComponent death = new DeathComponent(damageSource, attackerRef);
commandBuffer.addComponent(npcRef, DeathComponent.getComponentType(), death);

// The scheduler will now ensure NPCSystems.OnDeathSystem is executed for this entity,
// which will in turn add a DeferredCorpseRemoval component.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of a system. The framework manages their lifecycle.
  ```java
  // ANTI-PATTERN: Do not do this.
  NPCSystems.AddedSystem badSystem = new NPCSystems.AddedSystem(NPCEntity.getComponentType());
  badSystem.onEntityAdded(ref, reason, store, cmd); // This bypasses the scheduler and will cause undefined behavior.
  ```
- **Stateful Systems:** Do not add fields to a system class to store temporary data. Systems must remain stateless. Pass data between systems using components on entities.
- **Ignoring the CommandBuffer:** Modifying a component retrieved directly from the `Store` will break determinism and cause severe concurrency bugs.
  ```java
  // ANTI-PATTERN: Direct mutation.
  TransformComponent transform = store.getComponent(ref, TransformComponent.getComponentType());
  transform.setPosition(new Vector3f(0, 1, 0)); // WRONG. This is a race condition.

  // CORRECT: Use the CommandBuffer.
  commandBuffer.setComponent(ref, newTransform);
  ```

## Data Pipeline

The systems in NPCSystems are nodes in a larger data processing pipeline orchestrated by the ECS scheduler. Data flows from game events or state changes, through these systems, and results in queued commands for future state changes.

**Example Flow: NPC Initialization**
> Flow:
> Spawner Logic -> Creates Entity with `NPCEntity` component -> End of Tick -> ECS Scheduler detects new entity matching `AddedSystem` query -> **`NPCSystems.AddedSystem.onEntityAdded`** is invoked -> System queues commands on `CommandBuffer` (e.g., `addComponent(NewSpawnComponent)`) -> Scheduler finishes tick -> `CommandBuffer` is flushed -> `NewSpawnComponent` is now present on the NPC entity.


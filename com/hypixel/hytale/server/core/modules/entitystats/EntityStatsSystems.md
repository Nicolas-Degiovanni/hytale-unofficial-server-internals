---
description: Architectural reference for EntityStatsSystems
---

# EntityStatsSystems

**Package:** com.hypixel.hytale.server.core.modules.entitystats
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class EntityStatsSystems {
    // Contains only static inner classes
}
```

## Architecture & Concepts
The EntityStatsSystems class is not a concrete system that is registered with the engine's scheduler. Instead, it serves as a static namespace and logical grouping for a suite of highly-specialized ECS (Entity Component System) systems that collectively manage the entire lifecycle of entity statistics on the server.

These inner systems form a coordinated data processing pipeline that executes each game tick. The pipeline is responsible for:
1.  **Modification:** Applying changes to stats from passive effects like regeneration (Regenerate).
2.  **Reaction:** Triggering gameplay events when stats cross critical thresholds, such as an entity's death when health reaches zero (Changes).
3.  **Synchronization:** Propagating stat updates to clients for entities they can see (EntityTrackerUpdate).
4.  **Cleanup:** Resetting transient state at the end of the tick (ClearChanges).

The execution order of these systems is strictly controlled by ECS dependencies to ensure a deterministic and predictable outcome each tick. For example, modifications are always calculated before reactions are triggered, and reactions are processed before the final state is sent to clients.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. It is a static container for other system classes. The inner system classes (e.g., Changes, Regenerate) are instantiated by the EntityStatsModule during server bootstrap.
- **Scope:** The contained system definitions are available for the entire server lifetime. The instances of these systems persist for the entire server session.
- **Destruction:** The system instances are destroyed and de-registered during server shutdown.

---

## EntityStatsSystems.Changes

**Type:** Transient (ECS System)

### Definition
```java
// Signature
public static class Changes extends EntityTickingSystem<EntityStore> {
```

### Architecture & Concepts
The Changes system is the primary reactive component in the entity stats pipeline. It runs after all stat-modifying systems have completed their work for the current tick. Its sole responsibility is to inspect the final state of each entity's stats and trigger consequential game logic.

This system acts as a critical bridge between raw numerical data (the stat values) and concrete gameplay events. It is responsible for detecting when a stat reaches its configured minimum or maximum value. Upon detection, it initiates predefined InteractionChains, which can trigger a wide array of effects like applying status effects, playing sounds, or spawning particles.

Most importantly, this system contains the canonical logic for entity death. It specifically monitors the health stat and, if it falls to or below the minimum, adds a DeathComponent to the entity, initiating the death process.

### Lifecycle & Ownership
- **Creation:** Instantiated by the EntityStatsModule and registered with the world's system scheduler.
- **Scope:** A single instance exists for the lifetime of the server.
- **Destruction:** De-registered and garbage collected on server shutdown.

### Internal State & Concurrency
- **State:** This system is entirely stateless. All data is read from the components of the entity being processed in the current tick.
- **Thread Safety:** This system is **not thread-safe** and is explicitly marked to run serially (`isParallel` returns false). This is a critical design choice, as it queues interactions and adds components via a CommandBuffer, operations which must be ordered and cannot suffer from race conditions.

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(N) | Executes once per entity matching the query. N is the number of stats on the entity. This method is invoked exclusively by the ECS scheduler. |

### Integration Patterns

#### Standard Usage
This system is not invoked directly by developers. It is automatically registered and ticked by the engine. Its behavior is configured through asset files that define the min/max value effects for each EntityStatType.

#### Execution Order
The dependency configuration is critical for engine stability.
- **`Order.AFTER` StatModifyingSystem:** Guarantees that it reacts to the final, fully calculated stat values for the tick.
- **`Order.BEFORE` EntityTrackerUpdate:** Ensures that gameplay effects (like death) are processed *before* the final state is networked to clients, preventing visual inconsistencies.

### Data Pipeline
> Flow:
> EntityStatMap (updated by other systems) -> **Changes System** -> InteractionManager (queues effects) OR CommandBuffer (adds DeathComponent)

---

## EntityStatsSystems.EntityTrackerUpdate

**Type:** Transient (ECS System)

### Definition
```java
// Signature
public static class EntityTrackerUpdate extends EntityTickingSystem<EntityStore> {
```

### Architecture & Concepts
This system is the network synchronization layer for entity stats. Its purpose is to efficiently communicate stat changes from the server to all relevant connected clients. It integrates tightly with the EntityTrackerSystems, which determine which entities are visible to which players.

The system employs two modes of synchronization:
1.  **Full Synchronization:** When an entity becomes newly visible to a player, this system creates a complete snapshot of all its stats and sends it to that player's client. This ensures the entity appears on the client with the correct initial state.
2.  **Delta Synchronization:** For entities that are already visible, the system only sends changes that have occurred during the current tick. It uses dirty flags on the EntityStatMap component to identify which stats need updating, minimizing network bandwidth.

A distinction is made between updates for the entity's owner ("self") and other observers, as some stats may be private.

### Lifecycle & Ownership
- **Creation:** Instantiated by the EntityStatsModule and registered with the world's system scheduler.
- **Scope:** A single instance exists for the lifetime of the server.
- **Destruction:** De-registered and garbage collected on server shutdown.

### Internal State & Concurrency
- **State:** This system is stateless. It reads dirty flags from the EntityStatMap and writes to per-player network queues managed by the EntityTrackerSystems.
- **Thread Safety:** This system is designed to be parallelizable (`maybeUseParallel`). This is safe because each task operates on a distinct entity and writes to a thread-safe network queue associated with a specific viewer. There is no shared mutable state between parallel executions of the tick method.

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(V) | Executes once per entity matching the query. V is the number of viewers for the entity. Invoked exclusively by the ECS scheduler. |

### Integration Patterns

#### Standard Usage
This system operates automatically. Developers trigger its behavior implicitly by modifying an EntityStatMap component on a tracked entity.

#### Anti-Patterns (Do NOT do this)
- **Manual Packet Creation:** Never attempt to manually create and send EntityStatUpdate packets. This will bypass the tracking system and lead to client desynchronization. All stat changes must flow through the EntityStatMap component to be correctly synchronized by this system.

### Data Pipeline
> Flow:
> EntityStatMap (dirty flags are set) -> **EntityTrackerUpdate System** -> EntityViewer.queueUpdate -> ComponentUpdate Packet -> Network Layer -> Client

---

## EntityStatsSystems.Regenerate

**Type:** Transient (ECS System)

### Definition
```java
// Signature
public static class Regenerate<EntityType extends LivingEntity> extends EntityTickingSystem<EntityStore> implements EntityStatsSystems.StatModifyingSystem {
```

### Architecture & Concepts
The Regenerate system is a core gameplay mechanic implementation that applies passive regeneration or degeneration effects to entities over time. It is a concrete example of a StatModifyingSystem, a special interface that marks it for execution early in the tick's processing order.

Each tick, this system iterates through entities with stats. For each stat, it calculates the total regeneration amount by summing contributions from two sources:
1.  The entity's base regeneration values.
2.  Regeneration values provided by equipped armor or other items.

The system accounts for the time delta (dt) to ensure regeneration is frame-rate independent. It also contains logic to prevent regeneration from exceeding the stat's maximum value or falling below its minimum.

### Lifecycle & Ownership
- **Creation:** Instantiated by the EntityStatsModule and registered with the world's system scheduler.
- **Scope:** A single instance exists for the lifetime of the server.
- **Destruction:** De-registered and garbage collected on server shutdown.

### Internal State & Concurrency
- **State:** The system itself is stateless, but it performs mutations on the EntityStatMap component. It uses a transient field, tempRegenerationValues, on the component to aggregate regeneration values before applying the final sum.
- **Thread Safety:** This system is **not thread-safe** and is marked to run serially (`isParallel` returns false). Modifying the same EntityStatMap component from multiple threads would introduce race conditions and non-deterministic behavior.

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(S + I) | Executes once per entity. S is the number of stats, I is the number of equipped items. Invoked exclusively by the ECS scheduler. |

### Integration Patterns

#### Standard Usage
This system is not called directly. Its behavior is configured entirely through asset data. To give an entity regeneration, you define RegeneratingValue properties in its entity asset file or on its armor items.

#### System Ordering
As a StatModifyingSystem, the ECS scheduler guarantees that Regenerate will run before systems like Changes and EntityTrackerUpdate. This is essential for the engine's logic: regeneration must be applied *before* the engine checks for death, and *before* the new health value is sent to the client.

### Data Pipeline
> Flow:
> Entity Asset (base regen values) + Item Assets (armor regen values) -> **Regenerate System** -> Calculation using TimeResource -> EntityStatMap.addStatValue (mutates component state)


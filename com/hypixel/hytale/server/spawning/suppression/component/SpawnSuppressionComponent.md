---
description: Architectural reference for SpawnSuppressionComponent
---

# SpawnSuppressionComponent

**Package:** com.hypixel.hytale.server.spawning.suppression.component
**Type:** Data Component

## Definition
```java
// Signature
public class SpawnSuppressionComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The SpawnSuppressionComponent is a data-only component within the server-side Entity-Component-System (ECS) architecture. Its sole purpose is to attach a spawn suppression rule identifier to an entity. In the context of Hytale's server, this component is typically attached to invisible volume entities that define a specific zone or area in the world.

This component does not contain any logic. It acts as a tag or a data payload that is read and interpreted by the SpawningPlugin and its associated systems. The core spawning logic queries the world for entities possessing this component to determine if a potential spawn event should be canceled or modified within the entity's bounds. The internal *spawnSuppression* string serves as a key to look up a specific set of suppression rules, allowing for different types of spawn control (e.g., "no_hostile_mobs", "cave_spawns_only").

It is a fundamental building block for level designers and scripters to control mob population and create safe zones or specialized encounter areas without modifying the global spawning configuration.

## Lifecycle & Ownership
- **Creation:** A SpawnSuppressionComponent is never instantiated directly. It is created and attached to an entity by the EntityStore, typically during world loading from a prefab definition or programmatically by a higher-level game system.
- **Scope:** The lifecycle of this component is strictly bound to the lifecycle of its parent entity. It persists as long as the entity exists within the world's EntityStore.
- **Destruction:** The component is destroyed automatically when its parent entity is removed from the world. It can also be removed from an entity explicitly via the EntityStore API, which would immediately remove its spawn-suppressing effect.

## Internal State & Concurrency
- **State:** The component's state is mutable, consisting of a single string field, *spawnSuppression*. This allows for dynamic changes to an area's spawning rules at runtime by modifying the component's data.
- **Thread Safety:** This component is **not thread-safe** and must not be accessed concurrently. As with all ECS components, it is designed to be managed by a single, authoritative system within the main server thread's game loop. Any modification or access from other threads requires external synchronization, which is a significant anti-pattern. All interactions should be scheduled as tasks to be executed on the main game tick.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component from the SpawningPlugin. |
| getSpawnSuppression() | String | O(1) | Returns the identifier for the active spawn suppression rule. |
| setSpawnSuppression(String) | void | O(1) | Sets or updates the identifier for the spawn suppression rule. |
| clone() | Component | O(1) | Creates a shallow copy of the component. This is used internally by the engine when duplicating entities. |

## Integration Patterns

### Standard Usage
Developers should not interact with this component directly. Logic should be implemented in a server-side system that queries for entities with this component and then acts upon them. The SpawningSystem is the canonical consumer.

```java
// Hypothetical system processing spawn suppression zones
for (Entity entity : world.queryEntities(SpawnSuppressionComponent.class)) {
    SpawnSuppressionComponent suppressor = entity.getComponent(SpawnSuppressionComponent.class);
    String rule = suppressor.getSpawnSuppression();
    
    // Apply logic based on the rule to nearby spawn points
    applySuppressionRule(entity.getPosition(), rule);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SpawnSuppressionComponent()`. Components must be managed by the EntityStore. Incorrect instantiation will result in an unmanaged component that is ignored by all engine systems.
- **Stateful Logic:** Do not add methods containing game logic to this class. Components must remain simple data containers. All logic must reside in systems.
- **Cross-Thread Access:** Reading or writing the *spawnSuppression* field from an asynchronous task or a different thread without proper scheduling will lead to data corruption and server instability.

## Data Pipeline
This component serves as a data source for the server's spawning pipeline. It does not process data itself.

> Flow:
> World Load -> Entity Prefab Deserialized -> **SpawnSuppressionComponent** attached to Entity -> Spawning System Tick -> Query for Entities with Component -> Rule Read -> Spawn Algorithm Modified/Suppressed


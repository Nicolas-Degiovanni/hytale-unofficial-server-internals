---
description: Architectural reference for NewSpawnComponent
---

# NewSpawnComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient Data Component

## Definition
```java
// Signature
public class NewSpawnComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The NewSpawnComponent is a fundamental component within Hytale's Entity-Component-System (ECS) architecture. It functions as a short-lived, stateful timer attached to an Entity immediately after it has been spawned into the world.

Its primary role is to flag an entity as "newly spawned," allowing various systems to apply temporary logic, such as spawn protection, temporary visual effects, or altered AI behavior. The component itself contains no logic; it is a pure data container whose state is mutated and evaluated by a dedicated System during the server's game loop tick. This design decouples the state (the remaining spawn window duration) from the behavior (what happens during or after that window), adhering to ECS principles.

The static method getComponentType indicates a reliance on a central registry, the EntityModule, for component type identification. This is a performance-oriented pattern that avoids runtime type reflection, using a pre-registered type handle instead.

### Lifecycle & Ownership
- **Creation:** Instantiated and attached to an Entity by a spawner system or world generation logic at the moment the Entity is created. The initial duration of the spawn window is provided via its constructor.
- **Scope:** The component's lifetime is intentionally brief, typically lasting only a few seconds. It is strictly bound to its parent Entity and is expected to be removed early in the Entity's total lifespan.
- **Destruction:** An external System, such as a SpawnSystem, is responsible for polling this component each tick. When the newSpawnWindowPassed method returns true, that System removes the component from the Entity. The Java Garbage Collector then reclaims its memory.

## Internal State & Concurrency
- **State:** The component's state is **mutable**. Its internal field, newSpawnWindow, is decremented on each call to newSpawnWindowPassed. It is designed to be a countdown timer.
- **Thread Safety:** This component is **not thread-safe**. It is designed to be exclusively owned and managed by the server's main game loop thread. Concurrent modification of the newSpawnWindow field from multiple threads will lead to race conditions and unpredictable timer behavior. All interactions must be synchronized with the main server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type identifier for this component from the EntityModule. |
| newSpawnWindowPassed(float dt) | boolean | O(1) | Decrements the internal timer by the delta time (dt). Returns true if the timer has expired. **Warning:** This method mutates the component's internal state. |
| clone() | Component | O(1) | Creates a new instance of the component with an identical timer value. Used for entity duplication and templating. |

## Integration Patterns

### Standard Usage
This component is intended to be processed by a System that queries for all entities possessing it. The System updates the timer each frame and removes the component upon expiration.

```java
// Conceptual example within a hypothetical SpawnSystem update method
for (Entity entity : world.getEntitiesWith(NewSpawnComponent.class)) {
    NewSpawnComponent spawnComponent = entity.getComponent(NewSpawnComponent.class);

    // Update the timer with the frame's delta time
    if (spawnComponent.newSpawnWindowPassed(deltaTime)) {
        // Timer expired, remove the component to transition the entity's state
        entity.removeComponent(NewSpawnComponent.class);
        // Potentially trigger other logic, like ending spawn protection
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Lifecycle Management:** Do not hold a reference to this component and manage its state outside of an ECS System. Its lifecycle is intrinsically tied to the entity and the game loop.
- **Multi-threaded Access:** Never call newSpawnWindowPassed from an asynchronous task or a different thread. All updates must occur on the main server thread to prevent state corruption.
- **State Reset:** This component is not designed to be reset. If a new spawn window is needed, the existing component should be removed and a new one should be added.

## Data Pipeline
The NewSpawnComponent facilitates a simple state transition for an entity, driven by the passage of game time.

> Flow:
> Entity Spawner creates Entity -> **NewSpawnComponent** is attached -> Game Loop Tick provides delta time -> Spawn System calls newSpawnWindowPassed -> Component returns true -> Spawn System removes **NewSpawnComponent** -> Entity is no longer "newly spawned"


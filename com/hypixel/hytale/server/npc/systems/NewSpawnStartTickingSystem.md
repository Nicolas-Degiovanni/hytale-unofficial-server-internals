---
description: Architectural reference for NewSpawnStartTickingSystem
---

# NewSpawnStartTickingSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class NewSpawnStartTickingSystem extends TickingSystem<EntityStore> {
```

## Architecture & Concepts
The NewSpawnStartTickingSystem implements a critical one-frame delay for newly created entities. In an Entity Component System (ECS) architecture, entities are often created and have components attached over the course of a single game tick. Running their primary logic systems immediately in the same tick can lead to race conditions or operations on partially initialized objects.

This system solves this problem by acting as a state transition gate. When an entity is spawned, it is marked with a **NonTicking** component via the static helper method queueNewSpawn. This prevents other systems from processing it. The NewSpawnStartTickingSystem then waits until the next tick, removes the **NonTicking** component, and effectively "activates" the entity, allowing it to participate in the main game loop.

Its execution is explicitly ordered to run *after* the StepCleanupSystem, ensuring that it operates within a well-defined phase of the game tick, after primary physics and state updates have concluded.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's core ECS framework during world initialization. The system's dependencies, such as the required ResourceType for its queue, are resolved and injected by the parent plugin loader (NPCPlugin).
- **Scope:** The lifecycle of a NewSpawnStartTickingSystem instance is bound to a single **EntityStore** instance. It persists for the entire duration of a game world session.
- **Destruction:** The system is destroyed and garbage collected when its associated **EntityStore** (the game world) is unloaded by the server.

## Internal State & Concurrency
- **State:** This system is externally stateful. The system class itself holds no mutable instance fields, but it orchestrates state changes within the shared **EntityStore**. Its primary state mechanism is the **QueueResource**, a world-scoped resource containing a list of entities pending activation. This queue is highly mutable, being populated by various spawner systems and drained exclusively by this system's tick method.
- **Thread Safety:** **This system is not thread-safe and must be confined to the main server thread.** All operations, including calls to the static queueNewSpawn method and the framework-driven tick method, modify the shared **QueueResource** without any locking mechanisms. Concurrent access will lead to collection corruption and unpredictable behavior.

## API Surface
The public contract is minimal, designed for interaction with the ECS framework and other systems, not for direct user manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(N) | Framework-invoked method. Processes all queued entities from the last tick. N is the number of entities spawned in the previous frame. |
| queueNewSpawn(reference, store) | static void | O(1) | The primary entry point for other systems. Adds a **NonTicking** component to an entity and schedules it for activation on the next tick. |
| getDependencies() | Set | O(1) | Returns system execution order dependencies to the ECS scheduler. |

## Integration Patterns

### Standard Usage
This system is not used directly. Other systems, such as entity spawners, use its static helper method to ensure newly created entities are initialized safely.

```java
// Inside another system, after creating an entity
Ref<EntityStore> newEntityRef = world.createEntity();
// ... attach other components to newEntityRef ...

// Finally, queue the entity to start ticking on the next frame.
// This prevents logic from running on a partially built entity.
NewSpawnStartTickingSystem.queueNewSpawn(newEntityRef, world.getStore());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NewSpawnStartTickingSystem()`. The ECS framework is responsible for creating and managing system instances.
- **Manual Ticking:** Never call the `tick` method directly. The server's system scheduler invokes it at the correct time, respecting the dependencies declared by `getDependencies`.
- **Multi-threaded Access:** Do not call `queueNewSpawn` from an asynchronous task or worker thread. All entity modifications must occur on the main server thread to prevent state corruption.

## Data Pipeline
The system manages a simple but critical data flow that transitions an entity from a latent to an active state.

> Flow:
> Entity Spawner System -> `queueNewSpawn()` -> Entity is tagged with **NonTicking** component -> Reference is added to **QueueResource** -> Next Game Tick -> **NewSpawnStartTickingSystem.tick()** -> **NonTicking** component is removed -> Entity becomes active in subsequent systems.


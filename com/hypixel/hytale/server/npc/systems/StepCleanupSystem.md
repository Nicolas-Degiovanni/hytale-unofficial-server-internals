---
description: Architectural reference for StepCleanupSystem
---

# StepCleanupSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class StepCleanupSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The StepCleanupSystem is a specialized, single-purpose system within the server-side Entity Component System (ECS) framework. Its primary function is to enforce the **Transient Component** design pattern for the StepComponent.

In Hytale's ECS architecture, some components act as transient signals or one-shot commands rather than persistent state. A StepComponent is likely added to an entity to command it to perform a single movement step. This system ensures that the component exists for only a single tick, preventing the entity from stepping continuously.

It operates as a janitorial or maintenance system. By querying for all entities possessing a StepComponent and immediately queueing its removal, it guarantees that the "step" signal is consumed and reset before the next game tick begins. Its position late in the execution order, as defined by its dependencies, is critical. This allows other systems to observe and react to the presence of a StepComponent *before* it is removed.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's core system registrar during world initialization. It is not intended for manual creation by gameplay logic developers.
- **Scope:** Session-scoped. An instance of StepCleanupSystem persists for the entire lifetime of the server world it is registered with.
- **Destruction:** The system is destroyed and garbage collected when its parent world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is stateless. It holds an immutable reference to the StepComponent type information but does not store or cache any data between ticks. Its behavior is idempotent and depends solely on the entities processed in a given tick.
- **Thread Safety:** This class is **not thread-safe** for direct, unmanaged calls. All execution is managed by the ECS system scheduler, which guarantees that the *tick* method is invoked in a safe context. The use of a CommandBuffer for component removal is a key concurrency pattern; modifications are deferred and executed in a synchronized batch, preventing race conditions and data corruption within the underlying EntityStore.

## API Surface
The public contract is defined by its role as an EntityTickingSystem. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(N) | For N entities with a StepComponent, queues a removal command for each. This is the core logic, executed by the system scheduler. |
| getQuery() | Query | O(1) | Returns a query that selects all entities that currently possess a StepComponent. |
| getDependencies() | Set | O(1) | Declares its position in the execution graph. It depends on RootDependency, ensuring it runs late in the tick cycle. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Instead, they leverage its behavior by adding a StepComponent to an entity. The system's automatic execution handles the cleanup.

```java
// In another system, an NPC is commanded to take a step.
// The developer adds the component, relying on StepCleanupSystem
// to remove it at the end of the tick.

commandBuffer.addComponent(npcEntity, new StepComponent(...));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new StepCleanupSystem()`. The system must be instantiated and registered by the server's world builder to be integrated into the game loop.
- **Manual Invocation:** Never call the *tick* method directly. Doing so bypasses the scheduler, the dependency graph, and the CommandBuffer synchronization points, which will corrupt entity state.
- **Stateful Logic:** Do not modify this system to hold state. Its design relies on being a stateless, idempotent cleanup utility.

## Data Pipeline
The system acts as a terminal point in the data flow for the StepComponent, ensuring its lifecycle is limited to a single tick.

> Flow:
> AI or Pathfinding System -> CommandBuffer.addComponent(StepComponent) -> **StepCleanupSystem** -> CommandBuffer.removeComponent(StepComponent) -> EntityStore Update


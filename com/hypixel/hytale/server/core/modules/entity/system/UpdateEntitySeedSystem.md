---
description: Architectural reference for UpdateEntitySeedSystem
---

# UpdateEntitySeedSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** System Component

## Definition
```java
// Signature
public class UpdateEntitySeedSystem extends DelayedSystem<EntityStore> {
```

## Architecture & Concepts
The UpdateEntitySeedSystem is a server-side maintenance process operating within the Entity Component System (ECS) framework. Its sole responsibility is to periodically trigger an update of the random seed associated with entities that possess an EntityStore component.

This system extends DelayedSystem, indicating that it does not execute on every game tick. Instead, it operates on a fixed, one-second interval. This design decouples a low-priority, periodic task from the main, performance-sensitive game loop. The actual logic for updating the seed is not contained within this class; it acts as a scheduled invoker, delegating the operation to the active World instance. This promotes a clean separation of concerns, where systems orchestrate *when* actions occur, and domain objects (like World) define *how* they occur.

## Lifecycle & Ownership
- **Creation:** This system is instantiated and registered by the server's core module loader during world initialization. It is not intended for manual creation by developers.
- **Scope:** The instance is scoped to a single server-side World. It persists for the entire lifetime of that world.
- **Destruction:** The system is garbage collected when its parent World is unloaded, typically during a server shutdown or when a world is explicitly destroyed. The ECS framework manages its entire lifecycle.

## Internal State & Concurrency
- **State:** This class is **stateless**. The one-second delay is an immutable property configured at construction time. All necessary data for its operation, such as the component Store, is passed as arguments to the delayedTick method.
- **Thread Safety:** This system is **not thread-safe** and must not be accessed concurrently. The parent ECS framework guarantees that the delayedTick method is invoked from a single, designated thread in a serialized manner. Adding mutable state to this class would violate this contract and introduce severe concurrency risks.

## API Surface
The public contract is defined by the DelayedSystem superclass. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| delayedTick(dt, systemIndex, store) | void | O(N) | **Framework Internal.** Executes the seed update logic. The complexity is dependent on the World.updateEntitySeed implementation. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. It is automatically registered with the server's ECS runner. The following example is for conceptual understanding of how the framework might register such a system.

```java
// Conceptual: How the framework registers the system
// This code would exist within a world or module initializer.
SystemRegistry registry = world.getSystemRegistry();
registry.register(new UpdateEntitySeedSystem());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new UpdateEntitySeedSystem()` in game logic. The framework is responsible for its creation and registration.
- **Manual Invocation:** Never call the `delayedTick` method directly. This bypasses the timing and scheduling logic of the DelayedSystem framework and can lead to unpredictable behavior and race conditions.
- **State Modification:** Do not extend this class to add mutable fields. Systems should remain stateless to ensure predictable execution within the ECS.

## Data Pipeline
This system acts as a time-based trigger rather than a step in a continuous data processing pipeline.

> Flow:
> ECS System Runner -> Internal Timer (1.0s elapsed) -> **UpdateEntitySeedSystem.delayedTick()** -> World.updateEntitySeed() -> EntityStore Component Mutation


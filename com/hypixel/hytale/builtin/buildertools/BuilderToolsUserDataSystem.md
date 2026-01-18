---
description: Architectural reference for BuilderToolsUserDataSystem
---

# BuilderToolsUserDataSystem

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** System Component

## Definition
```java
// Signature
public class BuilderToolsUserDataSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The BuilderToolsUserDataSystem is a reactive, event-driven component within the server-side Entity Component System (ECS) framework. Its sole responsibility is to enforce a data contract: every Player entity must possess a BuilderToolsUserData component.

This system acts as an automatic initializer. It operates by defining a highly specific query that targets entities which are players but do *not* yet have the BuilderToolsUserData component. When the ECS engine detects a new entity matching this query—typically when a player first joins the server—it triggers this system's logic. The system then attaches the missing component, ensuring that any subsequent game logic or other systems can safely assume its presence on any player entity.

This pattern decouples the component initialization from the player creation logic itself, promoting a clean, single-responsibility architecture. The system is stateless and idempotent; its logic only runs once per entity to bring it into compliance with the data contract.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's central SystemRegistry during the world initialization phase. Systems are part of the core engine infrastructure and are not created on-demand.
- **Scope:** Session-scoped. A single instance of this system exists for the entire lifetime of the game world it is registered with.
- **Destruction:** The instance is destroyed and eligible for garbage collection only when the associated game world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is **stateless**. It contains no mutable fields and does not cache any entity data. Its behavior is determined entirely by its static, immutable Query object and the state of the entities passed to it by the ECS engine.

- **Thread Safety:** **Not thread-safe**. Like most ECS systems, it is designed to be executed serially within the main server game loop. Its methods, such as onEntityAdd, are invoked by the ECS engine and must not be called from other threads. All entity modifications performed by this system are assumed to be part of a single-threaded, sequential update tick to prevent data corruption and race conditions with other systems.

## API Surface
The public methods of this class are hooks designed for invocation by the ECS engine, not for direct use by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static query used by the ECS engine to identify target entities. |
| onEntityAdd(holder, reason, store) | void | O(1) | Engine callback. Attaches the BuilderToolsUserData component to the entity. |
| onEntityRemoved(holder, reason, store) | void | O(1) | Engine callback. This is a no-op and performs no action. |

## Integration Patterns

### Standard Usage
This system is not intended to be used directly. It is automatically registered and managed by the server's ECS engine. Its functionality is implicitly applied to all player entities as they are created. Developers should rely on the *effect* of this system—the guaranteed presence of the BuilderToolsUserData component on players—rather than interacting with the system itself.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderToolsUserDataSystem()`. The ECS engine is solely responsible for the lifecycle of its systems.
- **Manual Invocation:** Do not call `onEntityAdd` or other methods directly. Doing so bypasses the engine's query and state management, which can lead to inconsistent entity states or runtime exceptions.

## Data Pipeline
The system acts as a simple, one-way data transformer, enriching an entity's component set based on a predefined rule.

> Flow:
> Player Entity Creation -> ECS Engine Query Match -> **BuilderToolsUserDataSystem.onEntityAdd** -> Component Addition -> Modified Player Entity State


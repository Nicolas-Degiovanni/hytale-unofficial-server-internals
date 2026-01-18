---
description: Architectural reference for EnsureBuilderTools
---

# EnsureBuilderTools

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** ECS System

## Definition
```java
// Signature
public static class EnsureBuilderTools extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
EnsureBuilderTools is a reactive system within the server-side Entity-Component-System (ECS) framework. Its sole responsibility is to provision a standard, default set of "builder tools" to any entity that is identified as a Player. It functions as a policy enforcer, guaranteeing that upon creation or addition to the world, every player entity receives a clean, pre-configured toolkit.

The system operates on a subscription model, using its `getQuery` method to register interest exclusively in entities possessing the Player component. This is a classic observer pattern implementation, allowing the system to remain decoupled from the code that creates players. When the ECS engine detects that a new entity matches this query, it automatically invokes the `onEntityAdd` callback on this system.

Crucially, the specific items granted are not hardcoded. The system reads from `BuilderToolItemReferenceAsset` configurations, making the entire toolset data-driven. This allows designers and developers to modify the default builder tools by changing asset files, without requiring any code modifications.

## Lifecycle & Ownership
-   **Creation:** Instantiated automatically by the server's central `SystemRegistry` or an equivalent ECS bootstrap mechanism during server startup. This system is not designed for manual instantiation.
-   **Scope:** The system exists as a singleton-like instance within the context of a running game world. It persists for the entire lifetime of the server process.
-   **Destruction:** Decommissioned and garbage collected only when the server shuts down and the parent `SystemRegistry` is cleared.

## Internal State & Concurrency
-   **State:** This system is entirely **stateless**. It holds no instance-level fields and does not cache data between invocations. All operations are performed using the `Holder` and `Store` objects passed directly into its callback methods, ensuring predictable and idempotent behavior.
-   **Thread Safety:** **Not thread-safe.** This system is designed to be executed synchronously by the main server game loop or a dedicated system execution thread. All interactions with components, especially mutable ones like Inventory and ItemContainer, must be externally synchronized by the calling engine thread. Direct invocation from unmanaged threads will lead to severe race conditions and data corruption.

## API Surface
The public contract is defined by its implementation of the `HolderSystem` interface. These methods are callbacks intended for invocation by the ECS engine, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns a query that matches all entities with a Player component. This defines the system's subscription. |
| onEntityAdd(holder, reason, store) | void | O(N) | Callback triggered when a Player entity is added. Clears the player's tool inventory and populates it with items defined in assets. N is the number of builder tool items. |
| onEntityRemoved(holder, reason, store) | void | O(1) | A no-op callback. This system takes no action when a Player entity is removed from the world. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. It is registered with the server's ECS framework at startup and operates automatically in the background. The following example conceptually illustrates the event chain that triggers its logic.

```java
// This system is registered automatically by the engine.
// The following is a conceptual representation of the event that triggers it.

// 1. A player connects, and the engine creates an entity for them.
Entity playerEntity = world.createEntity();

// 2. The Player component is added to the entity. This action is detected
//    by the ECS engine, which then dispatches an "onAdd" event.
//    EnsureBuilderTools receives this event and executes its logic.
playerEntity.addComponent(new Player(...)); 
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new EnsureBuilderTools()`. The system's lifecycle is strictly managed by the engine's `SystemRegistry` to guarantee proper ordering and execution.
-   **Manual Invocation:** Calling `system.onEntityAdd(...)` from application code is a critical error. This bypasses the ECS framework's state management and event queue, which will lead to unpredictable behavior, state corruption, and potential crashes.
-   **System Overriding:** Extending this class to alter its behavior is not recommended. To change the tools given to players, modify the `BuilderToolItemReferenceAsset` files instead.

## Data Pipeline
The system acts as a processor in a simple, event-driven pipeline. It consumes an entity creation event and produces a side effect: a modified player inventory.

> Flow:
> Player Connects -> ECS Engine Creates Entity -> ECS Engine Adds Player Component -> SystemRegistry Dispatches Event -> **EnsureBuilderTools.onEntityAdd** -> Reads `BuilderToolItemReferenceAsset` -> Modifies Player.Inventory Component


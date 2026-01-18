---
description: Architectural reference for DiedInPortalSystem
---

# DiedInPortalSystem

**Package:** com.hypixel.hytale.builtin.portals.systems.curse
**Type:** System

## Definition
```java
// Signature
public class DiedInPortalSystem extends DeathSystems.OnDeathSystem {
```

## Architecture & Concepts
The DiedInPortalSystem is a reactive, event-driven system within the server-side Entity-Component-System (ECS) architecture. It is designed to handle a very specific edge case: a player's death while they are inside a special "Portal World".

This class subscribes to entity death events by extending the core `DeathSystems.OnDeathSystem`. The engine automatically invokes its logic when a `DeathComponent` is added to any entity matching its query (specifically, a Player entity).

Its primary responsibilities are:
1.  To check if the death occurred within the context of an active Portal World.
2.  If so, to flag the player's UUID in the `PortalWorld` resource, effectively recording that they died there.
3.  To trigger a cleanup of any "Cursed" items from the player's inventory, a gameplay mechanic tied to the portal dimension.

It acts as a specialized bridge, connecting the generic, engine-level death mechanics to the specific gameplay rules of the Portals feature.

### Lifecycle & Ownership
-   **Creation:** Instantiated automatically by the server's ECS System Scheduler during world initialization. This is not a class that developers instantiate manually.
-   **Scope:** The instance is stateless and its lifecycle is managed by the engine. It is invoked by the scheduler during the world update tick whenever a relevant component change occurs.
-   **Destruction:** The instance is destroyed when the game world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This system is **entirely stateless**. It contains no member fields and does not cache any data between invocations. All state is read from and written to the ECS world via method parameters like `Store`, `CommandBuffer`, and `Ref`.
-   **Thread Safety:** This system is thread-safe within the confines of the Hytale ECS scheduler. The engine guarantees that systems are executed in a deterministic order and that all mutations performed via the `CommandBuffer` are deferred and synchronized safely. Direct invocation of its methods from outside the engine's scheduler is unsupported and will lead to race conditions.

## API Surface
The public API is designed for consumption by the ECS engine, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(...) | void | O(1) | Engine callback invoked when a `DeathComponent` is added to a Player. Contains the core logic for the system. |
| getQuery() | Query | O(1) | Returns the component query used by the engine to identify which entities this system should operate on. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Its functionality is triggered implicitly by game events. The system is automatically registered with the engine and will execute its logic when a player dies.

To "use" this system, one must simply ensure it is included in the server's active system configuration. The triggering event is a player entity receiving a `DeathComponent`.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new DiedInPortalSystem()`. The ECS scheduler is solely responsible for its lifecycle.
-   **Manual Invocation:** Do not call the `onComponentAdded` method directly. This bypasses the engine's scheduling and command buffer synchronization, which will corrupt world state and cause severe concurrency bugs.

## Data Pipeline
This system processes data in response to a change in the ECS world state. The flow is unidirectional and reactive.

> Flow:
> Player takes fatal damage -> Core `DeathSystems` adds `DeathComponent` to Player entity -> ECS Scheduler dispatches component change -> **DiedInPortalSystem.onComponentAdded** is invoked -> System reads `PortalWorld` resource -> System writes Player UUID to `PortalWorld` -> System issues commands to delete `CursedItems` from Player inventory.


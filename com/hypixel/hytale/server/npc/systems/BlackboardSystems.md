---
description: Architectural reference for BlackboardSystems
---

# BlackboardSystems

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Utility

## Definition
```java
// Signature
public class BlackboardSystems {
    // Contains static inner System classes
}
```

## Architecture & Concepts

The **BlackboardSystems** class is a non-instantiable utility container that groups together several critical Entity Component Systems (ECS) responsible for integrating the AI **Blackboard** with the server's world simulation. It is not a system itself, but rather a logical namespace for the systems that manage the **Blackboard** resource's lifecycle and its reaction to world events.

In Hytale's server architecture, the **Blackboard** acts as a central, world-specific data repository for AI decision-making. It tracks entities, environmental states, and other contextual information required by NPCs. The systems within **BlackboardSystems** serve as the primary bridge between the core game engine's event stream and this AI-centric data store.

-   **Event Handling Systems** (**BreakBlockEventSystem**, **DamageBlockEventSystem**): These systems subscribe to specific world events. When an event occurs, the ECS framework dispatches it to the corresponding system, which then translates the event data and updates the **Blackboard**. This decouples the AI data layer from the direct cause of the event.
-   **Lifecycle Management System** (**InitSystem**): This system hooks into the creation and destruction of an **EntityStore** (representing a game world). It ensures that the **Blackboard** resource is properly initialized when a world loads and safely torn down when it unloads.
-   **Maintenance System** (**TickingSystem**): This system provides a periodic, low-frequency tick to the **Blackboard**, allowing it to perform background maintenance tasks such as cleaning up stale data or refreshing cached views, independent of the main game loop's high-frequency updates.

## Lifecycle & Ownership

-   **Creation:** The inner system classes (**InitSystem**, **TickingSystem**, etc.) are not instantiated directly. They are instantiated by the server's ECS framework during the bootstrap phase of an **EntityStore**. The outer **BlackboardSystems** class is never instantiated.
-   **Scope:** An instance of each inner system persists for the entire lifetime of the **EntityStore** to which it is attached. This scope is typically equivalent to the lifetime of a single game world on the server.
-   **Destruction:** The systems are destroyed and garbage collected when the parent **EntityStore** is torn down, for example, when a world is unloaded from memory. The **onSystemRemovedFromStore** callback in **InitSystem** is a critical part of this cleanup process, ensuring the **Blackboard** resource is properly finalized.

## Internal State & Concurrency

-   **State:** The outer **BlackboardSystems** class is stateless. The inner system classes are effectively stateless, holding only an immutable **ResourceType** handle. The state they operate on is stored externally within the **Blackboard** resource, which is a highly mutable, world-scoped object.
-   **Thread Safety:** **CRITICAL WARNING:** These systems are **not thread-safe**. They are designed to be executed by the single-threaded ECS scheduler that governs each **EntityStore**. All method calls (**handle**, **delayedTick**, **onSystemAddedToStore**) must occur on the world's main thread. Accessing these systems from other threads will lead to race conditions, data corruption, and server instability.

## API Surface

The public API is defined by the methods overridden from the base ECS system classes. Direct invocation is an anti-pattern.

### BreakBlockEventSystem & DamageBlockEventSystem

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(...) | void | O(1) | Forwards the block event to the world's **Blackboard** resource. Relies on the ECS framework for invocation. |
| getQuery() | Query | O(1) | Returns an empty query, indicating this system processes global events rather than iterating over specific entities. |

### InitSystem

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onSystemAddedToStore(store) | void | O(N) | Initializes the **Blackboard** resource when the system is added to a world. Complexity depends on world initialization logic. |
| onSystemRemovedFromStore(store) | void | O(N) | Triggers cleanup of the **Blackboard** resource when the world is unloaded. |

### TickingSystem

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| delayedTick(dt, index, store) | void | O(N) | Called periodically (every 5 seconds) by the ECS scheduler to trigger maintenance tasks on the **Blackboard**. |

## Integration Patterns

### Standard Usage

Developers do not interact with these systems directly. They are registered with the server's ECS configuration at startup. The engine then automatically manages their lifecycle and invocation. The primary interaction is indirect: causing a world event will trigger the corresponding system.

```java
// This code is conceptual and resides within the engine's bootstrap logic.
// A developer would NOT write this.

// During server world initialization...
EntityStore worldStore = createNewWorldStore();
Blackboard.ResourceType blackboardResource = Blackboard.getResourceType();

// The engine registers the systems.
worldStore.addSystem(new BlackboardSystems.InitSystem(blackboardResource));
worldStore.addSystem(new BlackboardSystems.TickingSystem(blackboardResource));
worldStore.addSystem(new BlackboardSystems.BreakBlockEventSystem());
worldStore.addSystem(new BlackboardSystems.DamageBlockEventSystem());
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BlackboardSystems.TickingSystem()` in game logic. The systems are managed entirely by the ECS framework. Manual creation will result in a system that is not registered and will never be executed.
-   **Manual Invocation:** Never call methods like **handle** or **delayedTick** directly. This bypasses the ECS scheduler, breaks the execution order, and can cause severe state corruption.
-   **State Storage:** Do not add mutable fields to these system classes. They are intended to be stateless logic containers. All state should reside in ECS Components or Resources like the **Blackboard**.

## Data Pipeline

The systems act as processors in a larger event-driven data pipeline. For an event system, the flow is unidirectional from a world event to an AI data structure update.

> Flow:
> Player Action (e.g., breaking a block) -> Server Game Logic -> **BreakBlockEvent** created and dispatched -> ECS Event Bus -> **BlackboardSystems.BreakBlockEventSystem.handle()** -> **Blackboard.onEntityBreakBlock()** -> Internal AI data structures are updated


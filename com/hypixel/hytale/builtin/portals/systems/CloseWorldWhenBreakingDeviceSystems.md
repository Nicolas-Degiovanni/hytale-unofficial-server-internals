---
description: Architectural reference for CloseWorldWhenBreakingDeviceSystems
---

# CloseWorldWhenBreakingDeviceSystems

**Package:** com.hypixel.hytale.builtin.portals.systems
**Type:** Utility

## Definition
```java
// Signature
public final class CloseWorldWhenBreakingDeviceSystems {
    // Private constructor to prevent instantiation

    public static class ComponentRemoved extends RefChangeSystem<ChunkStore, PortalDevice> {
        // ... system implementation
    }

    public static class EntityRemoved extends RefSystem<ChunkStore> {
        // ... system implementation
    }
}
```

## Architecture & Concepts

The CloseWorldWhenBreakingDeviceSystems class is a container for two distinct but related server-side systems within the Hytale Entity Component System (ECS) framework. It is not a system itself but rather a logical grouping for systems that manage the lifecycle of portal-linked worlds.

Its primary architectural role is **automated resource management**. Specifically, it ensures that dynamically created "fragment worlds"—worlds accessible only through a portal—are automatically unloaded and removed from memory when they are no longer accessible and contain no players.

This is a reactive system that responds to two key events:
1.  The removal of a PortalDevice component from an entity.
2.  The removal of an entire entity that possesses a PortalDevice component.

By hooking into these core ECS lifecycle events, this system prevents orphaned worlds from accumulating on the server, which would otherwise lead to significant memory and CPU consumption over time. It is a critical component for maintaining server stability in a universe with player-created, transient worlds.

### Lifecycle & Ownership
-   **Creation:** The nested system classes, ComponentRemoved and EntityRemoved, are not instantiated directly by developers. They are instantiated by the server's ECS System Scheduler during its initialization phase when systems are registered.
-   **Scope:** The lifetime of a system instance is managed entirely by the ECS framework. An instance may be created to process a batch of events for a single game tick and then be eligible for garbage collection, or it may persist for the lifetime of the server. The implementation details are abstracted by the framework.
-   **Destruction:** Managed automatically by the Java Garbage Collector when the ECS framework releases its references to the system instances. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** These systems are **stateless**. They do not contain any fields or cache any data. All necessary state (the entity reference, the component data, the world store) is provided as method parameters by the ECS framework during event processing. This stateless design is fundamental to the scalability and predictability of the ECS architecture.

-   **Thread Safety:** These systems are **not thread-safe** and must not be invoked from arbitrary threads. They are designed to be executed exclusively by the main server thread for a given world as part of the game tick's synchronous update loop. The CommandBuffer parameter is a key part of this model; it ensures that any mutations to the world state are deferred and executed at a safe point later in the tick, preventing race conditions and state corruption.

## API Surface

The public API is defined by the base classes from the ECS framework. The systems override specific methods to implement their logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentRemoved(...) | void | O(1) | Framework callback. Invoked when a PortalDevice component is removed from an entity. Triggers the world closure check. |
| onEntityRemove(...) | void | O(1) | Framework callback. Invoked when an entity with a PortalDevice component is removed from the world. Triggers the world closure check. |

## Integration Patterns

### Standard Usage

A developer does not call methods on these systems directly. Instead, the systems are registered with the server's System Scheduler, typically during server startup. The framework then automatically invokes them when relevant events occur.

```java
// Hypothetical server initialization code
SystemScheduler scheduler = server.getSystemScheduler();

// The framework handles instantiation and invocation
scheduler.register(CloseWorldWhenBreakingDeviceSystems.ComponentRemoved.class);
scheduler.register(CloseWorldWhenBreakingDeviceSystems.EntityRemoved.class);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CloseWorldWhenBreakingDeviceSystems.ComponentRemoved()`. The systems rely on the context and parameters provided by the ECS framework for execution. Manual instantiation serves no purpose and will fail if methods are called.
-   **Manual Invocation:** Never call the `onComponentRemoved` or `onEntityRemove` methods directly. Doing so bypasses the ECS scheduler, the CommandBuffer, and all safety guarantees of the game tick, almost certainly leading to state corruption or a server crash.
-   **External World Closure:** Do not attempt to replicate this logic in other game systems. Centralizing the responsibility for fragment world cleanup in this dedicated system prevents conflicting logic and ensures a single source of truth for resource management.

## Data Pipeline

The data flow for this system is event-driven, originating from an in-game action and culminating in a potential change to the server's universe state.

> Flow:
> In-Game Action (e.g., player breaks a portal block) -> Entity/Component is marked for removal -> **ECS Framework** detects change at end of tick -> Dispatches event to **CloseWorldWhenBreakingDeviceSystems** -> System checks `world.getPlayerCount()` -> If zero, dispatches `removeWorld` command to **Universe** -> Universe unloads world chunks and removes the world instance.


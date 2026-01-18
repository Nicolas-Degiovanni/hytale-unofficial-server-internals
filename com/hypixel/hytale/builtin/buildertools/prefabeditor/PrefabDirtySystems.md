---
description: Architectural reference for PrefabDirtySystems
---

# PrefabDirtySystems

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Utility

## Definition
```java
// Signature
public final class PrefabDirtySystems {
    // Inner classes defined below
}

// Inner System Signature 1
public static class BlockBreakDirtySystem extends EntityEventSystem<EntityStore, BreakBlockEvent> {
    // ...
}

// Inner System Signature 2
public static class BlockPlaceDirtySystem extends EntityEventSystem<EntityStore, PlaceBlockEvent> {
    // ...
}
```

## Architecture & Concepts
The PrefabDirtySystems class is a stateless utility that serves as a container for two critical Entity Component System (ECS) event listeners: BlockBreakDirtySystem and BlockPlaceDirtySystem.

Architecturally, this class acts as an **adaptor layer** between the core world simulation and the specialized Builder Tools plugin. Its sole responsibility is to translate low-level world modification events (a block being placed or broken) into a higher-level concept relevant to the prefab editor: the "dirty" state of a prefab.

By subscribing to `BreakBlockEvent` and `PlaceBlockEvent`, these systems decouple the core game engine from the builder tools. The block placement logic does not need any knowledge of prefabs or editing sessions. Instead, these systems observe the event bus and notify the appropriate manager, `PrefabEditSessionManager`, when a relevant change occurs within the bounds of an active editing session. This is a classic implementation of the Observer pattern within Hytale's ECS framework.

The systems themselves are extremely lightweight, containing no logic beyond delegating to the centralized `markDirtyAtPosition` method, which in turn queries the `PrefabEditSessionManager`.

## Lifecycle & Ownership
- **Creation:** The inner system classes, BlockBreakDirtySystem and BlockPlaceDirtySystem, are not instantiated directly by developers. They are instantiated by the Hytale ECS framework when the BuilderToolsPlugin is loaded and its systems are registered with the world. The outer PrefabDirtySystems class is a final utility with a private constructor and is never instantiated.
- **Scope:** Instances of the inner systems persist for the lifetime of the world they are registered in. They are active as long as the Builder Tools plugin is enabled for that world.
- **Destruction:** The systems are destroyed and garbage collected when the world is unloaded or the server shuts down, as part of the standard ECS cleanup process.

## Internal State & Concurrency
- **State:** This class and its inner systems are **stateless**. They do not cache data or maintain any internal state between invocations. All state management is delegated to the `PrefabEditSessionManager`.
- **Thread Safety:** The systems are **not thread-safe** and must only be executed by the main ECS processing thread. The `handle` methods delegate to `PrefabEditSessionManager`, which is not designed for concurrent access. Invoking these systems from multiple threads will lead to race conditions and corrupt the state of active prefab editing sessions.

## API Surface
The public API consists solely of the two inner system classes, which are intended for registration with the ECS, not for direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BlockBreakDirtySystem | class | N/A | An ECS system that listens for BreakBlockEvent. |
| BlockPlaceDirtySystem | class | N/A | An ECS system that listens for PlaceBlockEvent. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class or its systems directly. The `BuilderToolsPlugin` is responsible for registering these systems with the ECS engine upon initialization. Their operation is entirely implicit and event-driven. A developer working with prefabs would interact with the `PrefabEditSessionManager` to check the dirty state, not these systems.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockBreakDirtySystem()`. The ECS framework manages the lifecycle of these systems. Manual instantiation will result in a non-functional object that is not registered to receive any events.
- **Direct Invocation:** Do not call the `handle` method on a system instance. This bypasses the ECS scheduler and event bus, breaking the intended data flow and causing unpredictable behavior.
- **Manual Registration:** Avoid manually registering these systems. Rely on the `BuilderToolsPlugin` to manage their registration to ensure correct initialization order and dependencies.

## Data Pipeline
The data flow is unidirectional and triggered by player actions that modify the world. The systems act as a simple routing mechanism from the global event bus to the specific subsystem responsible for prefab state.

> Flow:
> Player Action (Place/Break Block) -> Core Engine -> **`PlaceBlockEvent` / `BreakBlockEvent`** -> ECS Event Bus -> **BlockPlaceDirtySystem / BlockBreakDirtySystem** -> `PrefabEditSessionManager` -> `PrefabEditSession` State Update (marked as dirty)


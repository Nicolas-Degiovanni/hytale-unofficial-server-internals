---
description: Architectural reference for PrefabPathHelper
---

# PrefabPathHelper

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Utility

## Definition
```java
// Signature
public final class PrefabPathHelper {
```

## Architecture & Concepts
The PrefabPathHelper is a stateless utility class that serves as a high-level factory for creating and configuring patrol path markers within the game world. It operates as a command-like function, encapsulating the multi-step process of entity creation and component attachment into a single, atomic-behaving operation.

Its primary architectural role is to abstract the complexities of the Entity Component System (ECS) from its callers, which are typically command parsers or server-side logic handlers. Instead of manually instantiating an entity, spawning it, and then attaching and configuring four separate components, a caller can use this helper to perform the entire sequence with one method call. It acts as a "recipe" for constructing a specific, well-defined entity type.

This class is a pure-logic container and holds no state. It directly manipulates the world state via the provided `Store<EntityStore>` object.

### Lifecycle & Ownership
- **Creation:** As a static utility class with a private constructor, PrefabPathHelper is never instantiated. It is loaded by the JVM's class loader at runtime.
- **Scope:** The class and its static method are available for the entire application lifetime after being loaded.
- **Destruction:** The class is unloaded only when the application shuts down and its class loader is garbage collected.

## Internal State & Concurrency
- **State:** This class is completely stateless. It contains no member fields and all required data is passed as arguments to its static method. All operations are performed on the externally managed state contained within the `Store`.
- **Thread Safety:** **NOT THREAD-SAFE.** The `addMarker` method performs direct, unsynchronized write operations on the `Store<EntityStore>`. The ECS `Store` is the canonical state of the game world and is not designed for concurrent access. All calls to this helper must be marshaled to the main server thread that is responsible for the world tick. Failure to do so will result in race conditions, data corruption, and unpredictable server crashes.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addMarker(store, playerRef, pathId, pathName, pauseTime, obsvAngleDegrees, targetIndex, worldgenId) | void | O(1) | Creates, spawns, and configures a PatrolPathMarkerEntity in the world. This is a world-mutating operation. Throws NullPointerException if the player's TransformComponent is missing. |

## Integration Patterns

### Standard Usage
The `addMarker` method should be invoked from server-side logic that has access to the world's primary `Store` and a valid entity reference, typically for a player executing a command.

```java
// Example from a server-side command handler
Store<EntityStore> worldStore = server.getWorld().getStore();
Ref<EntityStore> playerRef = commandContext.getPlayer().getReference();
UUID currentPathId = pathManager.getActivePathId();

// All parameters are provided by the command context
PrefabPathHelper.addMarker(
    worldStore,
    playerRef,
    currentPathId,
    "CavePatrol_01",
    2.5,
    90.0f,
    (short) 3,
    42
);
```

### Anti-Patterns (Do NOT do this)
- **Off-Thread Execution:** **CRITICAL:** Never call `addMarker` from an asynchronous task, network thread, or any thread other than the main world-tick thread. This will bypass the engine's state management guarantees and corrupt the ECS `Store`.
- **Invalid References:** Passing a stale or invalid `playerRef` will cause an assertion failure or `NullPointerException` when the helper attempts to retrieve the player's `TransformComponent`. Always ensure the reference is valid before calling.
- **Reflection-based Instantiation:** The constructor is private for a reason. Do not attempt to create an instance of `PrefabPathHelper` using reflection; it provides no benefit and violates the stateless utility pattern.

## Data Pipeline
PrefabPathHelper orchestrates a "write" pipeline that transforms a set of parameters into a fully realized entity within the world state.

> Flow:
> Command Parameters -> **PrefabPathHelper.addMarker** -> World.spawnEntity() -> Store.putComponent(Transform) -> Store.putComponent(Model) -> Store.putComponent(DisplayName) -> Store.putComponent(Nameplate) -> New Entity in World

The process is as follows:
1.  Input parameters defining the marker are received.
2.  The player's current position and rotation are read from the `Store` using the `playerRef`.
3.  A new `PatrolPathMarkerEntity` is instantiated in memory.
4.  The `World` service is called to spawn the entity at the player's location, which officially adds it to the `EntityStore` and assigns it a valid `Ref`.
5.  A series of `putComponent` calls are made to the `Store`, associating the new entity's `Ref` with fully configured instances of `ModelComponent`, `DisplayNameComponent`, and `Nameplate`.
6.  The final result is a new, visible, and fully configured path marker entity within the simulation.


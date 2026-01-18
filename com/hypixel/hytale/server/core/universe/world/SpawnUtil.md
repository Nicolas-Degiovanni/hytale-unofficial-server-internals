---
description: Architectural reference for SpawnUtil
---

# SpawnUtil

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Utility

## Definition
```java
// Signature
public final class SpawnUtil {
```

## Architecture & Concepts
SpawnUtil is a stateless utility class that centralizes the critical server logic for positioning entities within the world. It serves as a high-level orchestrator for manipulating an entity's spatial components, primarily the TransformComponent and HeadRotation. Its role is to abstract the underlying Entity-Component-System (ECS) operations related to spawning and teleportation into two distinct, high-level functions.

The class distinguishes between two fundamental scenarios:
1.  **Initial Spawn:** The `applyFirstSpawnTransform` method handles the creation and attachment of spatial components to an entity that does not yet exist in the world. It integrates with the world's configured `ISpawnProvider` to dynamically determine the correct spawn location, making the server's spawn behavior pluggable.
2.  **Teleportation:** The `applyTransform` method handles the repositioning of an entity that is already present in the world. It assumes the necessary spatial components exist and focuses solely on updating their state.

By providing this abstraction, SpawnUtil decouples game logic (like player connection handlers or command processors) from the implementation details of the ECS framework.

### Lifecycle & Ownership
-   **Creation:** As a final class with only static methods, SpawnUtil is never instantiated. It is loaded by the JVM ClassLoader upon its first use.
-   **Scope:** Application-level. Its methods are globally accessible throughout the server's runtime.
-   **Destruction:** The class is unloaded by the JVM during application shutdown. It manages no resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** SpawnUtil is completely stateless. It contains no member variables and all its methods operate exclusively on the arguments provided. The outcome of a method call is determined solely by its inputs.
-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the objects it manipulates, such as the `Holder<EntityStore>`, are mutable and are **not** guaranteed to be thread-safe. Callers are responsible for ensuring that any modifications to an entity's components via this utility are performed within a synchronized context, typically the main server thread, to prevent race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| applyFirstSpawnTransform(holder, world, config, uuid) | TransformComponent | O(1) | Calculates and applies the initial spawn transform. Creates and attaches new spatial components. Returns null if the world has no configured ISpawnProvider. |
| applyTransform(holder, transform) | void | O(1) | Updates an existing entity's position and rotation. Asserts that a TransformComponent already exists on the entity. |

## Integration Patterns

### Standard Usage
The primary use case is during player login to place the player entity into the world for the first time.

```java
// Example: Initializing a player entity on join
Holder<EntityStore> playerEntityHolder = ...;
World currentWorld = ...;
WorldConfig config = currentWorld.getConfig();
UUID playerUuid = ...;

// This creates and attaches the necessary components
TransformComponent tc = SpawnUtil.applyFirstSpawnTransform(
    playerEntityHolder,
    currentWorld,
    config,
    playerUuid
);

if (tc == null) {
    // CRITICAL: Handle server configuration error
    log.error("Failed to spawn player: No ISpawnProvider configured for world.");
    // Disconnect player or move to a fallback world
}
```

A secondary use case is for teleportation, such as via a game command.

```java
// Example: Teleporting an existing entity
Holder<EntityStore> playerEntityHolder = ...;
Transform destination = new Transform(new Vector3f(100, 64, 200), new Quaternion());

// This updates existing components
SpawnUtil.applyTransform(playerEntityHolder, destination);
```

### Anti-Patterns (Do NOT do this)
-   **Misusing `applyFirstSpawnTransform`:** Do not call `applyFirstSpawnTransform` on an entity that already has spatial components. This method is exclusively for initialization and will lead to a corrupt ECS state by adding duplicate components.
-   **Ignoring Null Return:** The `applyFirstSpawnTransform` method returns null if the world is misconfigured (lacks an `ISpawnProvider`). Failure to check for this null return will result in a NullPointerException downstream and indicates a critical server setup issue that must be handled gracefully.
-   **Violating Preconditions:** Do not call `applyTransform` on an entity that has not been initialized with a TransformComponent. The method contains an assertion that will crash the server in a development environment and cause undefined behavior in production. Always ensure an entity is fully spawned before attempting to teleport it.

## Data Pipeline
The flow of data through SpawnUtil is direct and transformational, converting high-level concepts like a player identity or a target location into low-level component data.

**Initial Spawn Pipeline:**
> Flow:
> Player UUID -> `WorldConfig` -> `ISpawnProvider` -> `Transform` -> **SpawnUtil.applyFirstSpawnTransform** -> New `TransformComponent` & `HeadRotation` -> Entity `Holder`

**Teleport Pipeline:**
> Flow:
> Target `Transform` -> **SpawnUtil.applyTransform** -> Updates existing `TransformComponent` & `HeadRotation` on Entity `Holder`


---
description: Architectural reference for DeployableTurretConfig
---

# DeployableTurretConfig

**Package:** com.hypixel.hytale.builtin.deployables.config
**Type:** Configuration / Data Object

## Definition
```java
// Signature
public class DeployableTurretConfig extends DeployableConfig {
```

## Architecture & Concepts
The DeployableTurretConfig class is a data-driven behavior controller for all turret-like deployable entities in the game. It is not merely a data container; it encapsulates the complete logic, state machine, and interaction patterns for turrets. This class is a prime example of Hytale's data-oriented design, where entity behavior is defined in configuration assets rather than hard-coded into engine systems.

Its core architectural pattern is a finite state machine implemented within the primary `tick` method. This state machine manages the turret's lifecycle from initialization to active combat:
*   **State 0 (Init):** A transient state that adds necessary ECS components like DeployableProjectileShooterComponent and transitions to the deploying state.
*   **State 1 (Start Deploy):** Triggers the initial deployment animation.
*   **State 2 (Await Deploy):** A waiting state that respects the `deployDelay` before the turret becomes active.
*   **State 3 (Attack):** The main operational state. It handles target acquisition, line-of-sight validation, rotation, and firing logic based on burst and cooldown timings.

This class operates entirely within the server-side Entity Component System (ECS). It reads entity state from the ECS `Store` (e.g., TransformComponent, HeadRotation) and queues state changes or new entity creation (e.g., projectiles) via the `CommandBuffer`. This ensures that all modifications are synchronized with the main game loop.

Configuration is loaded via the static `CODEC` field, which uses Hytale's reflection-based serialization system to populate an instance from an asset file. This allows designers to create diverse turret behaviors without modifying engine code.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly with `new`. They are instantiated and populated by the Hytale asset loader via the `BuilderCodec` system during server startup. The `afterDecode` hook calls `processConfig` to perform final setup, such as caching sound event indices.
-   **Scope:** A single instance of this class exists for each unique turret asset defined. It is loaded once and persists for the entire server session, acting as a shared, immutable template for all entities of its type.
-   **Destruction:** The object is garbage collected when the server shuts down or performs a full asset reload.

## Internal State & Concurrency
-   **State:** The DeployableTurretConfig object is **effectively immutable** after its initial creation and processing. All its fields represent static configuration values (e.g., `detectionRadius`, `ammo`, `shotInterval`). Runtime state, such as the current target, time since last shot, or remaining ammo, is **not** stored within this object. Instead, it is stored on the entity itself within ECS components like DeployableComponent and DeployableProjectileShooterComponent. This is a critical design choice that allows one config object to manage thousands of entities simultaneously without state conflicts.

-   **Thread Safety:** This class is **not thread-safe** and is designed to be operated exclusively by the main world update thread. The `tick` method reads from and writes to the ECS `Store` and `CommandBuffer`, which are not safe for concurrent access. Any asynchronous operations, such as applying knockback, are marshaled back to the main world thread using `world.execute()`, ensuring all game state mutations are properly synchronized.

## API Surface
The public contract is minimal, exposing only the `tick` method as the primary entry point for the engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(component, dt, index, chunk, store, buffer) | void | O(N) | The primary engine callback. Executes one cycle of the turret's state machine and logic. Complexity is O(N) where N is the number of entities within the `detectionRadius` during the target acquisition phase. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay programmers. It is automatically invoked by an underlying engine system (e.g., a `DeployableSystem`). The system iterates through all entities with a `DeployableComponent` and calls the `tick` method on the appropriate config instance associated with that entity.

A developer's interaction is limited to defining the turret's properties in a data file (e.g., JSON), which is then loaded by the `CODEC`.

```java
// Hypothetical engine code (system-level)
// This is NOT user code.
for (ArchetypeChunk<EntityStore> chunk : query) {
    for (int i = 0; i < chunk.size(); i++) {
        DeployableComponent deployable = chunk.getComponent(i, DeployableComponent.class);
        DeployableConfig config = deployable.getConfig(); // This would be an instance of DeployableTurretConfig

        // The engine dispatches the tick call to the correct config implementation
        config.tick(deployable, deltaTime, i, chunk, store, commandBuffer);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new DeployableTurretConfig()`. This bypasses the asset loading and codec system, resulting in a default-constructed, non-functional object. Turret behavior must be defined in asset files.
-   **Runtime State Modification:** Do not attempt to modify the fields of a cached `DeployableTurretConfig` instance at runtime. This is not thread-safe and will affect all turrets using that configuration, leading to unpredictable behavior.
-   **External Tick Invocation:** Calling the `tick` method from outside the main ECS update loop will break state consistency and cause severe concurrency violations.

## Data Pipeline
The `DeployableTurretConfig` acts as a processing node within the server's main game tick. It reads from the world state and writes commands to be executed at the end of the tick.

> Flow:
> Game Loop Tick -> `DeployableSystem` -> Query for Turret Entities -> **DeployableTurretConfig.tick()** -> Read `Store` (Entity Positions) -> Target Scan & LOS Logic -> Write `CommandBuffer` (Spawn Projectile, Update Components) -> End of Tick Sync Point


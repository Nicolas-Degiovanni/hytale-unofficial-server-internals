---
description: Architectural reference for DeployableTrapConfig
---

# DeployableTrapConfig

**Package:** com.hypixel.hytale.builtin.deployables.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class DeployableTrapConfig extends DeployableAoeConfig {
```

## Architecture & Concepts

The DeployableTrapConfig class is a data-driven configuration object that defines the behavior and logic for a server-side trap entity. It is a specialization of the more generic DeployableAoeConfig, adding trap-specific concepts like fuze times, active durations, and single-trigger destruction.

Architecturally, this class serves as a **behavioral template** within the Hytale Entity Component System (ECS). It is not an entity or a component itself. Instead, an instance of DeployableTrapConfig is deserialized from an asset file and referenced by one or more DeployableComponent instances. This pattern allows game designers to define numerous trap variations (e.g., spike traps, poison mines) in data files without requiring new code for each one.

The core of its functionality resides in the **tick** method, which is invoked by the server's core game loop for every active entity associated with this configuration. This method acts as a state machine, managing the trap's lifecycle from an initial "growing" phase to an armed "looping" state, and finally to its triggered or despawned state.

The static **CODEC** field is the primary mechanism for its creation, indicating that this class is tightly integrated with the engine's asset loading and serialization pipeline.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the engine's **Codec** system during server startup or when asset packs are loaded. The system deserializes a corresponding configuration file (e.g., a JSON asset) into a memory-resident DeployableTrapConfig object. Direct instantiation via a constructor is an invalid use case.
-   **Scope:** An instance of this class represents a single *type* of trap. It is loaded once and shared immutably across all entity instances of that trap type. Its lifetime is tied to the asset registry, persisting for the entire server session.
-   **Destruction:** The object is marked for garbage collection when the server shuts down or performs a hot-reload of its asset database. There is no manual destruction or cleanup method.

## Internal State & Concurrency

-   **State:** This object is effectively **immutable** after its initial creation from a configuration file. Fields such as fuzeDuration, activeDuration, and destroyOnTriggered are considered static configuration data and must not be modified at runtime. All dynamic, per-entity state (e.g., spawn time, current state flag, time since last attack) is stored within the **DeployableComponent** attached to the specific trap entity in the world. This separation of static configuration from dynamic state is a critical architectural principle.
-   **Thread Safety:** The class is **thread-safe for reads**. As its internal state is immutable, its configuration properties can be safely accessed from any thread. However, its state-mutating methods, particularly **tick**, are designed to be executed exclusively by the main server ECS thread. These methods operate on the world state via a **Store** and a **CommandBuffer**, which is the standard Hytale pattern for safely queuing world modifications from systems to be applied synchronously at the end of a tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(deployableComponent, dt, ...) | void | O(N) | The primary entry point called by the ECS each game tick. Manages the trap's state machine and triggers target detection. Complexity is O(N) where N is the number of entities within the detection radius. |
| handleDetection(store, cmdBuffer, ...) | void | O(N) | Overrides parent behavior to find and process valid targets within the trap's radius. Triggers damage and status effects on detected entities. |
| isLive(store, comp) | boolean | O(1) | Checks if the trap's fuze time has elapsed, making it active. Caches the result in the DeployableComponent's flags for efficiency. |
| onTriggered(store, ref) | void | O(1) | A callback invoked when the trap is triggered. Schedules the entity to be despawned after its activeDuration has passed. |

## Integration Patterns

### Standard Usage

A developer or system does not directly create or manage DeployableTrapConfig. The engine's DeployableSystem is responsible for iterating entities with a DeployableComponent and invoking the tick method on the appropriate config object referenced by that component.

```java
// Conceptual example of how the DeployableSystem would use this class.
// This code would exist within the engine, not in typical gameplay scripts.

// For each entity with a DeployableComponent...
DeployableComponent deployable = store.getComponent(entityRef, DeployableComponent.getComponentType());
DeployableConfig baseConfig = deployable.getConfig(); // This returns a DeployableTrapConfig instance

// The engine dispatches the tick call to our config object
if (baseConfig instanceof DeployableTrapConfig) {
    ((DeployableTrapConfig) baseConfig).tick(deployable, deltaTime, ...);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new DeployableTrapConfig()`. This bypasses the asset loading system and results in an unconfigured, non-functional object. All instances must be defined in asset files and loaded by the engine.
-   **Runtime State Modification:** Do not attempt to modify configuration fields like `fuzeDuration` at runtime. This is static data. To create a trap with a different fuze time, a new asset file must be created.
-   **External Invocation:** Do not call the `tick` method from outside the server's main ECS update loop. Doing so will bypass the CommandBuffer synchronization mechanism and lead to severe concurrency violations and world state corruption.

## Data Pipeline

The flow of configuration data and gameplay logic is distinct and follows a clear path.

> **Configuration Loading Flow:**
> Asset File (JSON) -> Engine Asset Loader -> **DeployableTrapConfig.CODEC** -> In-Memory **DeployableTrapConfig** Object

> **Gameplay Logic Flow:**
> Server Game Tick -> DeployableSystem -> Iterates Entities -> Gets DeployableComponent -> Calls **DeployableTrapConfig.tick()** -> TargetUtil Detection -> CommandBuffer Write (Damage/Despawn) -> Synchronized World State Update


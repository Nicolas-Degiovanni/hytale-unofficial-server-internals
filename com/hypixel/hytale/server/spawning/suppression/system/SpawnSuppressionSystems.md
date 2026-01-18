---
description: Architectural reference for SpawnSuppressionSystems
---

# SpawnSuppressionSystems

**Package:** com.hypixel.hytale.server.spawning.suppression.system
**Type:** Utility

## Definition
```java
// Signature
public class SpawnSuppressionSystems {
    // Contains only static nested System classes and private helper methods
}
```

## Architecture & Concepts

SpawnSuppressionSystems is a container class for a collection of highly specialized systems within the Hytale Entity Component System (ECS) framework. It is not a concrete object but rather a logical grouping for the core logic that prevents NPC spawning in designated areas. This mechanism is fundamental to gameplay design, allowing creators to establish safe zones, prevent over-population around points of interest, or dynamically alter spawning rules based on world events.

The architecture is event-driven and reactive, operating on entities that possess a **SpawnSuppressionComponent**. These systems act as the bridge between an entity's existence in the world and the low-level chunk data that the core spawning algorithms consult.

The primary components of this architecture are:

*   **Suppressor System:** The main actor. It watches for the creation and destruction of entities with a SpawnSuppressionComponent. Upon creation, it calculates a suppression volume (a radius around the entity) and "paints" all affected world chunks with suppression data. Upon destruction, it cleans up this data.
*   **Load System:** A lifecycle and asset management system. It handles the initial application of suppression rules when a world is loaded and, critically, manages the hot-reloading of SpawnSuppression asset files. If a game designer changes a suppression asset, this system ensures the changes are applied live to all worlds without a server restart by rebuilding the suppression maps.
*   **SpawnSuppressionController:** A world-scoped **Resource** that acts as the central, authoritative database for all active suppressors. It maintains a thread-safe map of suppressor entities to their properties and a map of chunk indices to their aggregate suppression state. The systems read from and write to this controller.
*   **ChunkSuppressionEntry:** A chunk-scoped **Component**. This is the final output of the suppression process. It is attached directly to a world chunk and contains a compact representation of all suppression zones overlapping that chunk. The core spawning logic performs a fast check against this component before attempting to place any new NPCs.

## Lifecycle & Ownership

The SpawnSuppressionSystems class itself is never instantiated. Its nested System classes follow the standard ECS lifecycle, managed by the SpawningPlugin.

-   **Creation:** Instances of the inner systems (Suppressor, Load, EnsureNetworkSendable) are created and registered with each World's EntityStore by the SpawningPlugin during server initialization or world creation.
-   **Scope:** Each system instance is scoped to a single World. It lives for the entire duration of that world's existence. A server with multiple active worlds will have multiple, independent instances of these systems.
-   **Destruction:** The systems are unregistered and garbage collected when a World is unloaded from the server. The Load system contains specific shutdown logic to deregister its asset event listeners from the global EventRegistry, preventing memory leaks.

## Internal State & Concurrency

The systems themselves are designed to be largely stateless. Their logic operates on the components of the entities they process and on shared, world-scoped Resources.

-   **State:** The authoritative state is held within the **SpawnSuppressionController** resource, not the systems. This resource uses a **Long2ObjectConcurrentHashMap** to store chunk suppression data, making it safe for modification from multiple threads. This is critical, as entity updates can occur concurrently.
-   **Thread Safety:** The systems are designed to be thread-safe within the context of the ECS framework. The Suppressor system's `onEntityAdded` and `onEntityRemove` methods may be called from different worker threads. All modifications to shared state are routed through the thread-safe SpawnSuppressionController or queued via the CommandBuffer for deferred execution by the engine. The Load system's event handlers execute tasks on the appropriate world thread to ensure safe access to world-specific data stores.

**WARNING:** Direct manipulation of the SpawnSuppressionController's internal maps from outside these systems is strictly forbidden and will lead to desynchronization, corrupted chunk data, and unpredictable spawning behavior.

## API Surface

The public contract is defined by the behavior of the nested ECS systems, not by traditional methods. Direct invocation is not intended.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Suppressor (System) | RefSystem | O(C\*log(N)) | Reacts to entities with SpawnSuppressionComponent. Adds or removes suppression data from the world. Complexity is proportional to the number of chunks (C) affected by the suppressor's radius. |
| Load (System) | StoreSystem | O(S) | Manages asset lifecycle. On asset load/reload, triggers a full rebuild of the suppression map for all active suppressors (S) whose asset definitions have changed. |
| EnsureNetworkSendable (System) | HolderSystem | O(1) | A utility system that guarantees any entity acting as a spawn suppressor is assigned a NetworkId, making it eligible for network synchronization to clients. |

## Integration Patterns

### Standard Usage

The intended use of this system is entirely declarative. A developer or designer defines an entity prefab and attaches a **SpawnSuppressionComponent** to it. The SpawningPlugin ensures the necessary systems are registered. When an instance of that prefab is placed in the world, the systems activate automatically.

```java
// This is a conceptual example. In practice, this is configured in asset files (e.g., JSON/HOCON).

// 1. An entity is created with the required components.
Entity mySuppressor = world.createEntity();
mySuppressor.addComponent(new TransformComponent(new Vector3d(100, 64, 100)));
mySuppressor.addComponent(new UUIDComponent(UUID.randomUUID()));

// 2. The key component that activates the suppression systems.
// "base_defense_field" would be an ID pointing to a SpawnSuppression asset file.
mySuppressor.addComponent(new SpawnSuppressionComponent("base_defense_field"));

// 3. The SpawnSuppressionSystems.Suppressor will now automatically process this entity
//    and apply the suppression rules defined in the asset to the surrounding chunks.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new SpawnSuppressionSystems.Suppressor()`. The systems must be instantiated and managed by the SpawningPlugin to be correctly integrated into the world's update loop and provided with the necessary resources.
-   **Manual State Management:** Do not directly access and modify the maps inside the SpawnSuppressionController resource. This bypasses the crucial logic that updates chunk components and can leave the world in a corrupted state.
-   **Ignoring Dependencies:** An entity with SpawnSuppressionComponent must also have a TransformComponent and a UUIDComponent for the system to function correctly. Missing these will result in runtime errors or silent failures.

## Data Pipeline

The flow of data demonstrates how an entity's presence is translated into a persistent rule affecting the game world.

> Flow:
> Entity with **SpawnSuppressionComponent** added to EntityStore -> **Suppressor System** (`onEntityAdded`) -> Read suppression asset, calculate affected chunks -> Update **SpawnSuppressionController** (central map) -> Queue updates via **ChunkSuppressionQueue** -> **ChunkStore** applies updates -> **ChunkSuppressionEntry** component is added/updated on affected Chunks -> Core **Spawning Algorithm** reads ChunkSuppressionEntry and blocks NPC spawn attempts.


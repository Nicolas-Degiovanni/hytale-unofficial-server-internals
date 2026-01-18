---
description: Architectural reference for SnapshotSystems
---

# SnapshotSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility

## Definition
```java
// Signature
public class SnapshotSystems {
    // Contains inner static classes: Add, Capture, Resize, SnapshotWorldInfo
}
```

## Architecture & Concepts

The SnapshotSystems class is a container for a group of related ECS (Entity Component System) systems responsible for capturing and storing an entity's historical transform data. This mechanism is fundamental for server-side features requiring knowledge of past entity states, such as network lag compensation, server-side replays, and advanced physics simulations.

This system does not operate in isolation; it is a core part of the server's entity processing pipeline. It acts as a data producer, systematically recording an entity's position and rotation each tick into a circular buffer.

The architecture is composed of four key parts, implemented as inner classes:
*   **SnapshotWorldInfo:** A world-scoped Resource that holds global configuration, such as the desired history length and the current world tick. This acts as the central source of truth for all snapshot operations.
*   **Add:** A `HolderSystem` that automatically attaches a `SnapshotBuffer` component to any new entity that possesses a `TransformComponent`. This ensures that any entity capable of movement is also capable of having its history recorded.
*   **Resize:** An `EntityTickingSystem` that runs early in the tick cycle. It detects changes in server configuration (e.g., tick rate) and resizes all `SnapshotBuffer` components accordingly. This preemptive action guarantees that buffers are correctly sized before new data is written.
*   **Capture:** The primary `EntityTickingSystem` that executes after `Resize`. It iterates through all relevant entities and copies their current position and rotation from their `TransformComponent` into their `SnapshotBuffer`, timestamping it with the current world tick.

## Lifecycle & Ownership

-   **Creation:** The systems within SnapshotSystems (`Add`, `Capture`, `Resize`) are discovered and instantiated by the ECS scheduler during the server's world initialization phase. The `SnapshotWorldInfo` resource is also created and registered with the world's `Store` at this time.
-   **Scope:** These systems and the associated resource persist for the entire lifetime of a server world. They are active from the moment the world is loaded until it is shut down.
-   **Destruction:** All system instances are discarded and the `SnapshotWorldInfo` resource is cleaned up when the server world is unloaded. The underlying `SnapshotBuffer` components are destroyed along with their parent entities.

## Internal State & Concurrency

-   **State:** The SnapshotSystems class itself is stateless and serves only as a namespace. The authoritative state is managed in two locations:
    1.  The global `SnapshotWorldInfo` resource, which holds mutable configuration data for the entire world.
    2.  The per-entity `SnapshotBuffer` components, which store the historical transform data.
-   **Thread Safety:** The systems are designed for parallel execution. The `isParallel` method allows the ECS scheduler to process different chunks of entities across multiple threads. Concurrency is managed through a strict dependency system:
    -   **Resize** is declared as a `RootDependency`, ensuring it runs at the beginning of the tick processing phase.
    -   **Capture** explicitly declares a dependency to run *after* `Resize`, preventing a race condition where a snapshot could be written to a buffer that is simultaneously being resized.

    **WARNING:** Modifying the static `HISTORY_LENGTH_NS` variable at runtime is not thread-safe and will lead to unpredictable behavior. This value should only be configured before server startup.

## API Surface

The primary API is not invoked directly but is instead consumed by the ECS scheduler. The key systems expose the standard `EntityTickingSystem` and `HolderSystem` contracts.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| **Add** (System) | `HolderSystem` | - | Reacts to entity creation. Ensures entities with a `TransformComponent` also receive a `SnapshotBuffer`. |
| **Resize** (System) | `EntityTickingSystem` | O(N) | Runs each tick to resize all `SnapshotBuffer` components if server configuration has changed. N is the number of entities with a snapshot buffer. |
| **Capture** (System) | `EntityTickingSystem` | O(N) | Runs each tick to capture the current transform of entities into their `SnapshotBuffer`. N is the number of entities with a snapshot buffer. |
| **SnapshotWorldInfo** (Resource) | `Resource` | O(1) | A world-scoped data object holding snapshot configuration. Accessed via the `Store`. |

## Integration Patterns

### Standard Usage

A developer does not interact with these systems directly. The behavior is triggered declaratively by the composition of an entity. To enable snapshotting for an entity, simply ensure it has a `TransformComponent`.

```java
// An entity is created with a TransformComponent
// The ECS framework handles the rest automatically.

// 1. SnapshotSystems.Add detects the new entity.
// 2. It adds a SnapshotBuffer component to the entity.
// 3. On subsequent ticks:
//    - SnapshotSystems.Resize ensures the buffer size is correct.
//    - SnapshotSystems.Capture records the transform into the buffer.

// To access the historical data, another system would query for the SnapshotBuffer.
SnapshotBuffer buffer = entity.getComponent(SnapshotBuffer.getComponentType());
Optional<Snapshot> pastState = buffer.findSnapshot(targetTick);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create instances of `SnapshotSystems.Add`, `Capture`, or `Resize` using `new`. The ECS framework is solely responsible for their lifecycle.
-   **Manual Component Management:** Do not manually add a `SnapshotBuffer` to an entity. The `Add` system is designed to manage this relationship automatically. Manually adding it can break initialization logic.
-   **Ignoring Dependencies:** When creating a new system that relies on up-to-date snapshot data, you **must** declare a dependency to run after `SnapshotSystems.Capture`. Failure to do so will result in reading stale, one-tick-old data.

## Data Pipeline

The flow of data is orchestrated by the server's main game loop and the ECS scheduler. For each tick, the pipeline is as follows:

> Flow:
> Game Tick Start -> **Resize System** (Checks config, resizes all buffers if needed) -> Other Physics/AI Systems (Update TransformComponent) -> **Capture System** (Reads final TransformComponent state) -> Data written to `SnapshotBuffer` -> Other Network/Logic Systems (Can now read the snapshot for the current tick) -> Game Tick End


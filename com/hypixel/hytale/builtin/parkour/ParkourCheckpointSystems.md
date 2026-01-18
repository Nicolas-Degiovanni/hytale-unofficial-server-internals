---
description: Architectural reference for ParkourCheckpointSystems
---

# ParkourCheckpointSystems

**Package:** com.hypixel.hytale.builtin.parkour
**Type:** Utility

## Definition
```java
// Signature
public class ParkourCheckpointSystems {
    public static class EnsureNetworkSendable extends HolderSystem<EntityStore> { ... }
    public static class Init extends RefSystem<EntityStore> { ... }
    public static class Ticking extends EntityTickingSystem<EntityStore> { ... }
}
```

## Architecture & Concepts
The ParkourCheckpointSystems class is a container for a collection of related systems within the Hytale Entity Component System (ECS) framework. It does not represent a single object instance but rather a logical grouping of systems that collectively manage the lifecycle and game logic of parkour checkpoints.

These systems are designed to be registered with the central ECS Store and are invoked by the engine's scheduler. They operate on entities possessing the ParkourCheckpoint component, handling everything from initial setup to player interaction during the game loop. The architecture strictly separates concerns into three distinct, coordinated systems.

### System 1: Init
The Init system is a reactive system that executes exactly once when a new entity with a ParkourCheckpoint component is added to the world. Its sole responsibility is to register the new checkpoint with the global ParkourPlugin singleton. It reads the checkpoint's index and the entity's UUID and populates the central tracking maps within the plugin. This ensures that the core game logic has a complete and accurate registry of all available checkpoints.

### System 2: EnsureNetworkSendable
This is a data integrity system. It guarantees that any parkour checkpoint entity can be synchronized with clients over the network. It queries for entities that have a ParkourCheckpoint component but are missing a NetworkId component. Upon detection, it immediately adds a new NetworkId, making the entity network-addressable. This preemptively resolves potential replication issues before they can occur.

### System 3: Ticking
The Ticking system contains the primary game logic and is executed every server tick. It queries for all active parkour checkpoints. For each checkpoint, it performs an efficient, proximity-based spatial query to find nearby players. If a player is detected within the checkpoint's activation radius, the system invokes the core `handleCheckpointUpdate` logic. This logic is responsible for managing player progress, tracking start and completion times, and sending feedback messages to the player.

**WARNING:** This system's performance is directly tied to the efficiency of the spatial query resource. A high density of players and checkpoints can increase its workload.

## Lifecycle & Ownership
- **Creation:** The nested system classes are instantiated by the server's bootstrap sequence, typically during the registration phase of the ParkourPlugin. They are not created on-demand.
- **Scope:** These systems are singletons within the context of the ECS Store. They persist for the entire lifetime of the server world they are registered with.
- **Destruction:** The systems are destroyed and garbage collected only when the parent ECS Store is shut down, for example, during a server stop or world unload.

## Internal State & Concurrency
- **State:** The systems themselves are stateless. They hold final references to ComponentType objects for efficient component access but do not store mutable data between invocations. All shared, mutable state is externalized to the ParkourPlugin singleton (e.g., `currentCheckpointByPlayerMap`, `startTimeByPlayerMap`).
- **Thread Safety:** The systems are designed to be operated on by the single-threaded ECS scheduler. Direct, multi-threaded invocation is unsupported and will lead to catastrophic state corruption. The Ticking system utilizes a `ThreadLocal` list for its spatial query results, ensuring that concurrent execution of multiple ECS worlds would not cause interference.

**WARNING:** All modifications to the shared state within ParkourPlugin must be thread-safe if the plugin is accessed from outside the ECS scheduler's thread.

## API Surface
The systems within ParkourCheckpointSystems have no public API intended for direct developer consumption. Their methods (`onEntityAdd`, `tick`, etc.) constitute a contract with the ECS framework and should only be invoked by the engine's scheduler.

## Integration Patterns

### Standard Usage
The systems are not used directly. They are registered with the ECS Store, typically via a central plugin or module manager. The engine then automatically invokes them at the appropriate times.

```java
// Example of how the systems are registered with the ECS
// This code would exist within the ParkourPlugin's initialization logic.

// Assume 'ecsStore' is the main world store
ecsStore.addSystem(new ParkourCheckpointSystems.Init(parkourCheckpointType));
ecsStore.addSystem(new ParkourCheckpointSystems.EnsureNetworkSendable());
ecsStore.addSystem(new ParkourCheckpointSystems.Ticking(parkourCheckpointType, playerType, playerSpatialResource));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Never create an instance of a system and call its methods manually. This bypasses the ECS scheduler, query system, and dependency management, leading to unpredictable behavior and state corruption.
- **Incorrect Registration Order:** The Ticking system has an explicit dependency on PlayerSpatialSystem. Registering systems in a way that violates this dependency will result in stale spatial data and broken checkpoint detection.
- **Modifying ParkourPlugin State Asynchronously:** Do not access and modify the state maps within ParkourPlugin from other threads without proper synchronization. The Ticking system assumes its access is synchronized by the game loop.

## Data Pipeline
The Ticking system is the most complex data processor in this group. Its data flow is critical to the parkour feature.

> Flow:
> Player Movement -> PlayerSpatialSystem (updates spatial index) -> **ParkourCheckpointSystems.Ticking** (queries spatial index for nearby players) -> ParkourPlugin (updates player progress state) -> Player.sendMessage -> Network Message -> Client UI Notification


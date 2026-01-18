---
description: Architectural reference for RepulsionSystems
---

# RepulsionSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.repulsion
**Type:** Utility

## Definition
```java
// Signature
public class RepulsionSystems {
```

## Architecture & Concepts

The RepulsionSystems class is a static container, or namespace, for a collection of tightly-related ECS systems that collectively implement the entity-repulsion feature on the server. This feature allows entities to exert a "push-away" force on other nearby entities, preventing unnatural clipping and creating more dynamic physical interactions.

This class does not represent a single system but rather a complete, self-contained module broken down into four distinct responsibilities, each handled by a nested static class:

-   **PlayerSetup:** A system responsible for initializing Player entities with a Repulsion component. It reads configuration from the world's gameplay settings to determine if and how players should repel others, attaching the appropriate component upon player creation.

-   **RepulsionTicker:** The core physics simulation system. On each tick, it queries the spatial partition for entities within an entity's repulsion radius. For each nearby entity, it calculates a force vector and applies it by writing a velocity change instruction to the ECS CommandBuffer. This system implements IVelocityModifyingSystem, signaling its role in the physics pipeline.

-   **EntityTrackerUpdate:** A networking system that synchronizes the state of the Repulsion component to clients. It integrates with the EntityTrackerSystems to efficiently send updates only when the repulsion configuration changes or when an entity becomes visible to a new client.

-   **EntityTrackerRemove:** A networking cleanup system. When a Repulsion component is removed from an entity, this system ensures a corresponding "remove" message is queued for all clients that can see the entity, keeping client-side state consistent.

Together, these systems form a complete vertical slice of functionality, from initial component setup and core game logic to network synchronization and cleanup.

### Lifecycle & Ownership
-   **Creation:** The RepulsionSystems class itself is never instantiated. The nested system classes (PlayerSetup, RepulsionTicker, etc.) are instantiated once by the server's core system loader during world initialization.
-   **Scope:** Instances of the nested systems persist for the entire lifetime of the server's World object. They are owned and managed by the central ECS runner.
-   **Destruction:** The systems are discarded and garbage collected only when the World they belong to is unloaded.

## Internal State & Concurrency
-   **State:** The RepulsionSystems container is stateless. The nested system classes are also effectively stateless; their fields are final references to immutable configuration objects like ComponentType and Query. All mutable state is read from and written to ECS components (e.g., TransformComponent, Velocity, Repulsion) within the Store and CommandBuffer.

-   **Thread Safety:** The systems are designed to operate within a concurrent ECS framework.
    -   **RepulsionTicker** performs read-only operations on the shared world state (Store, SpatialResource) and writes all mutations as deferred instructions to a CommandBuffer. This is a standard pattern to avoid race conditions and ensure deterministic updates.
    -   **EntityTrackerUpdate** explicitly supports parallel execution via its isParallel method. The engine can safely schedule its tick method to run concurrently across different batches of entities.
    -   **WARNING:** Direct, multi-threaded access to these systems is not supported and would violate the design of the ECS engine. All interaction must occur through the engine's main loop and CommandBuffer system.

## API Surface

The RepulsionSystems class itself exposes no public API. The nested classes implement standard engine system interfaces. Their primary purpose is to be registered with the engine, not called directly.

| System Class | Interface | Role |
| :--- | :--- | :--- |
| PlayerSetup | HolderSystem | Attaches Repulsion components to new Player entities based on world configuration. |
| RepulsionTicker | EntityTickingSystem | The primary physics logic loop. Calculates and applies repulsion forces between entities. |
| EntityTrackerUpdate | EntityTickingSystem | Synchronizes Repulsion component state to clients when it changes or an entity becomes visible. |
| EntityTrackerRemove | RefChangeSystem | Cleans up network state by notifying clients when a Repulsion component is removed. |

## Integration Patterns

### Standard Usage

These systems are not meant to be used directly. They are registered with the server's system graph during the world initialization process. The engine is then responsible for invoking their lifecycle methods at the correct time and in the correct order.

```java
// Example of how the engine would register these systems
SystemGraphBuilder<EntityStore> builder = ...;

// Physics and Setup
builder.add(new RepulsionSystems.PlayerSetup(...));
builder.add(new RepulsionSystems.RepulsionTicker(...));

// Networking
builder.add(new RepulsionSystems.EntityTrackerUpdate(...));
builder.add(new RepulsionSystems.EntityTrackerRemove(...));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually create instances of the nested systems post-initialization. They are part of the core engine loop and must be managed by it.
-   **Manual Invocation:** Never call the tick, onEntityAdd, or other system methods directly. Doing so will bypass the CommandBuffer, dependency ordering, and thread management provided by the ECS engine, leading to race conditions, corrupted state, and crashes.

## Data Pipeline

The systems create two primary data flows: one for physics simulation and one for network synchronization.

**Physics Simulation Flow:**
> Flow:
> **RepulsionTicker** (tick) -> SpatialResource Query -> Read **TransformComponent** (nearby entities) -> Calculate Force Vector -> Write to **Velocity** component (via CommandBuffer)

**Network Synchronization Flow:**
> Flow:
> **Repulsion** component state change -> **EntityTrackerUpdate** (tick detects change) -> Create ComponentUpdate message -> Queue message via **EntityTrackerSystems.EntityViewer** -> Network Packet



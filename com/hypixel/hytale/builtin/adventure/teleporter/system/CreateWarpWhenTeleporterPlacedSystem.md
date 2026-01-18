---
description: Architectural reference for CreateWarpWhenTeleporterPlacedSystem
---

# CreateWarpWhenTeleporterPlacedSystem

**Package:** com.hypixel.hytale.builtin.adventure.teleporter.system
**Type:** System

## Definition
```java
// Signature
public class CreateWarpWhenTeleporterPlacedSystem extends RefChangeSystem<ChunkStore, PlacedByInteractionComponent> {
```

## Architecture & Concepts
The CreateWarpWhenTeleporterPlacedSystem is a reactive, event-driven system within the Hytale Entity Component System (ECS) framework. Its sole responsibility is to bridge the gameplay action of a player placing a teleporter block with the creation of a persistent, named warp point in the global teleportation service.

This system operates as a side-effect handler. The primary event is a block placement, which results in a `PlacedByInteractionComponent` being attached to a block entity. This system subscribes to this specific component addition event. By using a query that also requires the `Teleporter` component, it ensures its logic only executes on entities that are designated as teleporters, not on every block placed by a player.

Upon activation, it extracts contextual information—such as the player who placed the block, their language settings, and the block's precise location and rotation—to generate a new `Warp` object. This object is then registered with the singleton `TeleportPlugin`, making the new location accessible via the server's global warp commands and UI.

## Lifecycle & Ownership
-   **Creation:** Instantiated automatically by the server's ECS System Manager during world initialization. This class is not designed for manual instantiation by developers.
-   **Scope:** The system's lifecycle is tied to the server or world session. A single instance exists and remains active until the world is unloaded or the server shuts down.
-   **Destruction:** The instance is destroyed and garbage collected as part of the world teardown process managed by the ECS framework.

## Internal State & Concurrency
-   **State:** This system is **stateless**. It contains no mutable instance fields and relies entirely on the data passed into its methods by the ECS framework. The `WARP_OFFSET` is a static final constant, ensuring it cannot be modified at runtime. This stateless design enhances predictability and robustness.

-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, its methods are invoked by the main world tick thread. Concurrency concerns are delegated to the components it interacts with:
    -   The `CommandBuffer` provides a deferred, thread-safe mechanism for component modification.
    -   The `TeleportPlugin` is a global singleton. Calls to `getWarps().put()` and `saveWarps()` must be internally synchronized or otherwise protected against race conditions if they can be accessed from other threads. It is assumed the ECS engine guarantees that system event handlers like `onComponentAdded` are executed serially within a single world tick.

    **Warning:** Developers must not invoke this system's methods from asynchronous tasks or other threads. All interaction must be mediated by the ECS framework to preserve world state integrity.

## API Surface
The primary API is the contract with the `RefChangeSystem` parent class, not direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(ref, placedBy, ...) | void | O(c) | Core logic handler. Triggered by the ECS engine when a matching component is added. Reads world state and creates a new warp via the TeleportPlugin. |
| createWarp(worldChunk, blockStateInfo, name) | static void | O(c) | Utility function to calculate the precise warp position and rotation based on the teleporter block's state and register it with the TeleportPlugin. |
| componentType() | ComponentType | O(1) | Returns the component type this system listens for, fulfilling the RefChangeSystem contract. |
| getQuery() | Query | O(1) | Defines the component filter. The system only processes entities with PlacedByInteractionComponent, Teleporter, and BlockStateInfo. |

## Integration Patterns

### Standard Usage
This system is not used directly. It is registered with the engine's system manager at startup. Its functionality is invoked implicitly by in-game actions. The "usage" is to ensure a block asset is configured with the correct components (`Teleporter`, `BlockStateInfo`) to allow this system's query to match it upon placement.

A hypothetical system registration might look like this:

```java
// In a world or plugin initialization routine
SystemManager systemManager = world.getSystemManager();
systemManager.register(new CreateWarpWhenTeleporterPlacedSystem());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CreateWarpWhenTeleporterPlacedSystem()` in gameplay logic. The system is managed by the engine.
-   **Manual Invocation:** Never call `onComponentAdded` directly. Doing so bypasses the ECS engine's state management, query filtering, and command buffering, which will lead to world corruption, race conditions, and unpredictable behavior. The framework is solely responsible for invoking this method with the correct context.
-   **Stateful Modifications:** Do not modify this class to hold state (e.g., caching the last player). Systems should remain stateless to ensure they are idempotent and predictable.

## Data Pipeline
The system acts as a processor in a larger data flow that begins with player input and ends with persistent world data modification.

> Flow:
> Player places a teleporter block -> Client sends "Place Block" packet -> Server validates and creates a block entity -> ECS attaches `PlacedByInteractionComponent` -> ECS framework matches the entity against the system's query -> **CreateWarpWhenTeleporterPlacedSystem.onComponentAdded** is invoked -> System reads player, block, and chunk data -> System calculates warp coordinates -> System calls `TeleportPlugin.get().getWarps().put()` -> `TeleportPlugin` updates its in-memory map and schedules a write to persistent storage.


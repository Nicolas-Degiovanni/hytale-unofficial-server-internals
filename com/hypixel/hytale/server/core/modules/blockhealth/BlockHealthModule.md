---
description: Architectural reference for BlockHealthModule
---

# BlockHealthModule

**Package:** com.hypixel.hytale.server.core.modules.blockhealth
**Type:** Singleton

## Definition
```java
// Signature
public class BlockHealthModule extends JavaPlugin {
```

## Architecture & Concepts

The BlockHealthModule is a server-side plugin responsible for managing the entire lifecycle of block damage and regeneration. It is a fundamental system for core gameplay mechanics such as mining, environmental destruction, and temporary block invulnerability after placement.

Architecturally, this module is a pure implementation of the Entity Component System (ECS) pattern. It does not maintain any direct state about individual blocks. Instead, it defines a component, **BlockHealthChunk**, and registers a series of systems that operate on entities possessing this component. The state is decentralized into `BlockHealthChunk` components, which are attached to each world chunk entity (`ChunkStore`).

The module's primary responsibilities are:
1.  **State Management:** Attaching a `BlockHealthChunk` component to every world chunk to store damage and fragility data for blocks within that chunk's boundaries.
2.  **Game Logic Execution:** Implementing the core logic for block health regeneration over time via the **BlockHealthSystem**. This system is ticked by the main server loop.
3.  **Event Handling:** Responding to in-game events, such as a player placing a block, to apply temporary fragility timers via the **PlaceBlockEventSystem**.
4.  **Network Synchronization:** Creating and dispatching `UpdateBlockDamage` packets to clients to visually represent block damage. This is handled by the **BlockHealthPacketSystem** when a chunk's data is requested by a player.

This design decouples the block health logic from other game systems. Any system can interact with block health by either querying for the `BlockHealthChunk` component or by emitting a relevant event.

### Lifecycle & Ownership

-   **Creation:** The `BlockHealthModule` is instantiated once by the server's plugin loader during the bootstrap sequence. The constructor immediately populates a static `instance` field, establishing its singleton status. Its `setup` method is then called, which registers all its associated components and systems with the server's ECS registries.

-   **Scope:** The module is a global singleton that persists for the entire runtime of the server. Its lifecycle is tied directly to the server's lifecycle.

-   **Destruction:** The object is dereferenced and eligible for garbage collection only during server shutdown when the plugin system is dismantled.

## Internal State & Concurrency

-   **State:** The `BlockHealthModule` class itself is effectively stateless after its initial setup. Its only internal field, `blockHealthChunkComponentType`, is initialized once and then becomes immutable. The actual mutable state (the health of every damaged block in the world) is stored within the `BlockHealthChunk` components distributed across all chunk entities.

-   **Thread Safety:** This module is designed for a concurrent environment.
    -   The `setup` method is called from a single thread during server initialization, making it inherently safe.
    -   The registered systems operate within the ECS framework's concurrency model. The **BlockHealthPacketSystem** explicitly declares `isParallel() = true`, indicating it is designed to be executed concurrently on multiple threads, likely for different players or chunks simultaneously.
    -   Systems like **BlockHealthSystem** are ticked by the main game loop. While they may read from shared data structures (e.g., the list of all players), all state modifications to the ECS are expected to be funneled through a `CommandBuffer`. This pattern defers writes to a synchronization point at the end of the tick, preventing race conditions.

    **WARNING:** Directly accessing and modifying a `BlockHealthChunk` component from an external thread without using the ECS CommandBuffer will corrupt world state and lead to severe concurrency bugs.

## API Surface

The public API of the module itself is minimal, as its primary function is to register systems. Interaction is intended to happen through the ECS, not direct method calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static BlockHealthModule | O(1) | Retrieves the global singleton instance of the module. Throws NullPointerException if called before the plugin is loaded. |
| getBlockHealthChunkComponentType() | ComponentType | O(1) | Returns the ECS component type definition for `BlockHealthChunk`. Essential for querying or modifying block health data within other systems. |

## Integration Patterns

### Standard Usage

Developers should not interact with this module directly. Instead, they interact with the data it manages through the ECS. For example, a custom weapon system would apply damage by retrieving the appropriate `BlockHealthChunk` and modifying its state via a `CommandBuffer`.

The module's own systems demonstrate the primary integration patterns: event listening and data processing.

**Listening to an Event:**
```java
// The PlaceBlockEventSystem listens for block placements
// and applies a fragility timer.
public void handle(..., PlaceBlockEvent event) {
    World world = commandBuffer.getExternalData().getWorld();
    Vector3i blockLocation = event.getTargetBlock();
    
    // Find the chunk and its BlockHealthChunk component
    long chunkIndex = ChunkUtil.indexChunkFromBlock(blockLocation.x, blockLocation.z);
    Ref<ChunkStore> chunkRef = world.getChunkStore().getChunkReference(chunkIndex);
    
    if (chunkRef != null) {
        BlockHealthChunk healthChunk = ...getComponent(chunkRef, ...);
        healthChunk.makeBlockFragile(blockLocation, ...);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new BlockHealthModule()`. The server's plugin loader is solely responsible for its creation. Always use `BlockHealthModule.get()` to retrieve the singleton instance.
-   **Premature Access:** Do not call `BlockHealthModule.get()` from another plugin's constructor. The initialization order is not guaranteed. Access it only after the plugin `setup` phase is complete.
-   **Stateful Systems:** Do not store references to `BlockHealthChunk` components inside other systems. Always query for the component within the system's execution method (`tick`, `handle`, etc.) to ensure you are operating on the most recent state.

## Data Pipeline

The module establishes several key data flows for managing block health.

**Block Regeneration Pipeline:**
> Flow:
> Server Tick -> **BlockHealthSystem.tick()** -> Read `TimeResource` -> Access `BlockHealthChunk` -> Calculate health regeneration -> Update `BlockHealth` map -> Create `UpdateBlockDamage` packet -> Send to visible players

**New Block Fragility Pipeline:**
> Flow:
> Player Action -> `PlaceBlockEvent` emitted -> Event Bus -> **PlaceBlockEventSystem.handle()** -> Find `BlockHealthChunk` for the position -> Add fragility data to the component's internal map

**Client Synchronization Pipeline (on chunk load):**
> Flow:
> Player enters new chunk -> Server requests chunk data for client -> **BlockHealthPacketSystem.fetch()** -> Read `BlockHealthChunk` -> Generate `UpdateBlockDamage` packets for all damaged blocks -> Packets are bundled with other chunk data and sent to client


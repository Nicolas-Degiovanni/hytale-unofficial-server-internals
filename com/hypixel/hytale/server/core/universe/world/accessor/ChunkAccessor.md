---
description: Architectural reference for ChunkAccessor
---

# ChunkAccessor

**Package:** com.hypixel.hytale.server.core.universe.world.accessor
**Type:** Interface / Contract

## Definition
```java
// Signature
@Deprecated
public interface ChunkAccessor<WorldChunk extends BlockAccessor> extends IChunkAccessorSync<WorldChunk> {
```

## Architecture & Concepts

The ChunkAccessor interface defines a high-level, server-side contract for interacting with a grid of world chunks. It serves as an abstraction layer, allowing various game systems to query and modify world data using absolute block coordinates without needing to manage the underlying chunk data structures directly. Its primary responsibility is to translate world-space requests into chunk-specific operations.

This interface is a fundamental component of the world simulation, providing the methods necessary to read block data (e.g., fluids) and, more importantly, to trigger updates that can propagate across chunk boundaries. The `performBlockUpdate` method is a key example, orchestrating a 3x3 neighborhood update to ensure that changes to one block correctly notify its neighbors, even if they reside in different chunks.

**WARNING:** This interface is marked as **Deprecated**. It should not be used for new development. Its existence in the codebase is likely for legacy support or as a precursor to a more refined world access pattern. Analysis of this interface is valuable for understanding the evolution of the engine's world architecture.

## Lifecycle & Ownership

As an interface, ChunkAccessor does not have its own lifecycle. The following describes the lifecycle of a typical *implementing object*, such as a `World` or `WorldView` class.

-   **Creation:** Concrete implementations are instantiated by the server's `Universe` manager when a world is loaded or created. It is not intended to be created directly by game logic systems.
-   **Scope:** The lifetime of an implementing object is tied directly to the lifetime of the world it represents. It persists as long as the world is loaded and active on the server.
-   **Destruction:** The object is eligible for garbage collection when the server unloads the corresponding world, ensuring all references to its chunks and other world data are released.

## Internal State & Concurrency

-   **State:** Any class implementing ChunkAccessor is expected to manage a highly mutable and complex state, typically involving a map or grid of loaded `WorldChunk` objects. This state is a live, in-memory representation of a portion of the game world.
-   **Thread Safety:** Implementations are **not thread-safe** and are designed to be accessed exclusively from the main server game loop thread. Unsynchronized access from other threads will lead to severe concurrency issues, including `ConcurrentModificationException`, data corruption, and server crashes. All world modifications must be scheduled and executed on the main thread.

## API Surface

The public contract consists of default methods that build upon the methods defined in the parent interface, `IChunkAccessorSync`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFluidId(x, y, z) | int | O(1) | Retrieves the fluid ID at the given world coordinates. Returns 0 if the chunk is not loaded. |
| performBlockUpdate(x, y, z) | boolean | O(1) | Triggers a block update in a 3x3x3 area around the target. Delegates to the more detailed overload. |
| performBlockUpdate(x, y, z, allowPartialLoad) | boolean | O(1) | Triggers a block update, flagging neighboring blocks for ticking. Returns false if any required chunks are not loaded. |

## Integration Patterns

### Standard Usage

The standard pattern involves obtaining the ChunkAccessor (typically by getting the `World` object) from a service context and using it to query or modify the world in response to a game event.

```java
// Example: A system handling block placement
// The 'world' object implements ChunkAccessor
ChunkAccessor world = server.getUniverse().getWorld(worldId);

// Place the block (logic not shown)
// ...

// Notify the 3x3 neighborhood of the change
boolean success = world.performBlockUpdate(x, y, z);
if (!success) {
    // Handle cases where neighboring chunks weren't loaded
    // This might involve rolling back the change or logging a warning
}
```

### Anti-Patterns (Do NOT do this)

-   **Usage in New Code:** The most significant anti-pattern is using this deprecated interface for any new feature development. A newer, non-deprecated alternative must be used.
-   **Ignoring Return Values:** The `performBlockUpdate` method returns a boolean indicating whether all neighboring chunks were available to receive the update. Ignoring this return value can lead to silent failures where block physics and interactions at chunk borders do not execute correctly.
-   **Cross-Thread Modification:** Never call methods like `performBlockUpdate` from a worker thread. All world modifications must be marshaled back to the main server thread to prevent data corruption.

## Data Pipeline

The `performBlockUpdate` method is a critical part of the game's data pipeline for world simulation. It acts as an initiator for the block ticking system.

> Flow:
> Game Event (e.g., Player breaks block) -> Event Handler -> **ChunkAccessor.performBlockUpdate** -> Flags 27 blocks with `setTicking(true)` -> World Tick System -> Processes ticked blocks -> State Change (e.g., water flows, sand falls) -> Network Packet sent to clients


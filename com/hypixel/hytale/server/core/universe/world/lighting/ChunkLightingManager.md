---
description: Architectural reference for ChunkLightingManager
---

# ChunkLightingManager

**Package:** com.hypixel.hytale.server.core.universe.world.lighting
**Type:** Service

## Definition
```java
// Signature
public class ChunkLightingManager implements Runnable {
```

## Architecture & Concepts
The **ChunkLightingManager** is a dedicated, asynchronous service responsible for calculating and propagating light within a single **World** instance. Its primary architectural purpose is to offload computationally expensive lighting calculations from the main server thread, preventing server ticks from lagging due to world modifications like block placement or destruction.

This manager operates as a classic producer-consumer system. External game systems (producers) submit chunks that require lighting updates to a work queue. A single, dedicated background thread owned by the manager (the consumer) processes this queue, performing the calculations and applying the results back to the world's chunk data.

A key design feature is the use of the Strategy pattern via the **LightCalculation** interface. This decouples the manager's orchestration logic from the specific lighting algorithm (e.g., **FloodLightCalculation**), allowing different lighting models to be implemented and swapped without altering the core processing loop.

## Lifecycle & Ownership
- **Creation:** A **ChunkLightingManager** is instantiated by its parent **World** object during the world's initialization phase. There is a strict 1:1 ownership model; each world has its own independent lighting manager.
- **Scope:** The instance persists for the entire lifetime of the **World** it serves. It is started via the **start** method, which spawns its dedicated processing thread.
- **Destruction:** The owning **World** is responsible for explicitly stopping the manager during the world shutdown sequence by calling the **stop** or **interrupt** method. The **stop** method includes a forceful termination mechanism (a call to the deprecated **Thread.stop**) if the thread does not shut down gracefully within 5 seconds, highlighting the critical importance of preventing orphaned lighting threads.

## Internal State & Concurrency
- **State:** The manager is highly stateful. Its core state consists of:
    - An **ObjectArrayFIFOQueue** named **queue** which holds the **Vector3i** coordinates of chunks pending calculation.
    - A thread-safe **Set** which mirrors the queue's contents to provide fast, O(1) duplicate checking.
    - A **Semaphore** used to efficiently put the worker thread to sleep when the queue is empty and wake it when new work arrives.
    - A reference to the active **LightCalculation** strategy.
- **Thread Safety:** This class is designed for high-concurrency environments. The public API, particularly **addToQueue**, is safe to be called from any thread (typically the main server thread). The internal state is protected through a combination of a **ConcurrentHashMap.newKeySet**, explicit **synchronized** blocks for queue access, and the **Semaphore** for thread signaling. The core processing loop in the **run** method is the single consumer, ensuring that light calculations for different chunks are serialized.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start() | void | O(1) | Spawns and starts the dedicated background thread for processing the lighting queue. |
| stop() | void | O(N) | Attempts a graceful shutdown of the worker thread, with a forceful timeout. |
| addToQueue(Vector3i chunkPosition) | void | O(1) | Thread-safe method to submit a chunk's coordinate for lighting recalculation. |
| invalidateLightAtBlock(...) | boolean | O(k) | High-level API to invalidate light around a specific block, delegating to the current **LightCalculation** strategy. |
| invalidateLightInChunk(WorldChunk chunk) | boolean | O(k) | Invalidates all light within a given **WorldChunk**, queuing all its sections for recalculation. |
| setLightCalculation(LightCalculation calc) | void | O(1) | Swaps the underlying lighting algorithm strategy. **WARNING:** This is not thread-safe and should only be done during initialization or controlled state changes. |

## Integration Patterns

### Standard Usage
The manager should be retrieved from the **World** instance. Game logic that modifies blocks should call the appropriate invalidation methods to trigger a lighting update.

```java
// Example: Block placement logic
World world = ...;
ChunkLightingManager lightingManager = world.getLightingManager();

// A block was placed, invalidating the light in the surrounding area.
// The light calculation strategy will determine which chunks to update.
lightingManager.invalidateLightAtBlock(worldChunk, x, y, z, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use **new ChunkLightingManager()**. Each **World** is responsible for creating and managing its own instance.
- **Manual Thread Management:** Do not call the **run** method directly. Use the **start** method to correctly initialize the background thread.
- **Unmanaged Lifecycle:** Failure to call **stop** or **interrupt** during world shutdown will result in a leaked thread, consuming resources and potentially causing instability.
- **State Tampering:** Do not attempt to modify the internal queue or set directly from outside the class. Use the provided public API.

## Data Pipeline
The flow of data for a lighting update is orchestrated by the manager but executed by the underlying strategy.

> Flow:
> Game Event (e.g., Block Break) → World Logic → **ChunkLightingManager.invalidate...()** → LightCalculation Strategy → **ChunkLightingManager.addToQueue()** → Internal Work Queue → Dedicated Lighting Thread → **LightCalculation.calculateLight()** → Direct modification of **BlockSection** light data arrays.


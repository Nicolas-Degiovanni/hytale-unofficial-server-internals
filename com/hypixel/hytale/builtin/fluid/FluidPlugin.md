---
description: Architectural reference for FluidPlugin
---

# FluidPlugin

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** Singleton

## Definition
```java
// Signature
public class FluidPlugin extends JavaPlugin {
```

## Architecture & Concepts

The FluidPlugin is a foundational server plugin responsible for bootstrapping and managing the entire fluid simulation system. It does not contain the simulation logic itself; rather, it acts as a central registry and coordinator that injects fluid-related behaviors into the core server's chunk processing pipeline.

Its primary architectural function is to integrate with three key engine systems during server startup:

1.  **Codec Registry:** It registers the concrete implementations of fluid behaviors, such as DefaultFluidTicker and FiniteFluidTicker. This allows the server to deserialize fluid definitions from asset files and associate them with the correct simulation logic.
2.  **Chunk Store Registry:** It registers a series of `FluidSystems`. These systems operate on chunk data and are responsible for the lifecycle of fluid information within a chunk, including data structure initialization, migration, network replication, and the invocation of the main simulation tick. This follows a data-oriented design, where logic is brought to the data (chunks).
3.  **Event Bus:** It subscribes to the ChunkPreLoadProcessEvent. This is the most critical integration, allowing the plugin to perform an intensive pre-simulation or "settling" phase on newly generated chunks. This ensures that fluids like oceans and rivers appear stable and correct the moment a player loads the area, preventing jarring, real-time simulation artifacts.

The plugin's design decouples the core engine from the specifics of fluid simulation. The engine only needs to know about the generic plugin and event systems, while the FluidPlugin provides the specialized implementation.

### Lifecycle & Ownership
-   **Creation:** A single instance of FluidPlugin is created by the server's plugin loader during the server bootstrap sequence. Its constructor is passed a JavaPluginInit context, which provides access to core server registries.
-   **Scope:** The instance is a singleton that persists for the entire runtime of the server. It is accessible globally via the static `get()` method.
-   **Destruction:** The instance is discarded and eligible for garbage collection during server shutdown when the plugin system unloads all active plugins.

## Internal State & Concurrency
-   **State:** The FluidPlugin class itself is effectively stateless. Its only instance field is the static singleton reference. All state related to fluid simulation is stored externally in world data, specifically within FluidSection and BlockSection components attached to chunk entities.
-   **Thread Safety:** The plugin is conditionally thread-safe, relying on the engine's concurrency model.
    -   The `setup` method is executed in a single-threaded context during server initialization.
    -   The primary logic within `onChunkPreProcess` is executed by the world's chunk processing thread pool. The event-driven model guarantees that each call to this handler operates on a distinct chunk, preventing data races on chunk data.
    -   **WARNING:** While the handler is safe within its own context, developers should not assume that accessing global state from within this handler is safe without proper synchronization. All operations should be confined to the chunk data provided by the event.

## API Surface

The public API is minimal, as the plugin's primary role is system integration, not direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static FluidPlugin | O(1) | Returns the global singleton instance of the plugin. |

## Integration Patterns

### Standard Usage

Direct interaction with the FluidPlugin instance is rare. The primary method of integration is data-driven: developers define new fluids and their behaviors in asset files. The systems registered by this plugin will automatically discover and process them.

In the rare case where access to the plugin instance is required, retrieve it from the static accessor.

```java
// Retrieve the singleton instance
FluidPlugin fluidPlugin = FluidPlugin.get();

// Note: There are no public methods to call for simulation.
// The plugin's value is in its setup and event handling.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new FluidPlugin()`. The server's plugin loader is solely responsible for its lifecycle. Manual instantiation will fail or lead to a broken, non-functional state.
-   **Manual Setup:** Do not invoke the `setup` method. It is part of the plugin lifecycle and is called automatically by the server. Calling it manually will result in duplicate registrations and will throw exceptions.

## Data Pipeline

The most significant data processing pipeline managed by this plugin is the pre-simulation of fluids in newly generated chunks. This pipeline ensures fluids are in a stable state before a player ever sees them.

> Flow:
> World Generator -> Creates new Chunk Data -> Engine fires **ChunkPreLoadProcessEvent**
>
> 1.  **Event Reception:** The `onChunkPreProcess` method receives the event for a single, newly generated chunk.
> 2.  **Data Traversal:** The method iterates through every block position within the chunk's FluidSection components.
> 3.  **Initial State Pruning:** For each fluid block, it performs a series of checks:
>     -   Is the fluid inside a solid block? If so, remove the fluid.
>     -   Can this fluid spread to an adjacent empty block? If not, and the fluid type cannot demote (e.g., become a source block), mark it as non-ticking to optimize performance.
>     -   Otherwise, mark the block as requiring a tick update.
> 4.  **Pre-Simulation Loop:** A `do-while` loop is initiated to simulate fluid flow for a maximum of 100 ticks or until the fluid state becomes stable (no more ticking blocks).
> 5.  **Ticker Invocation:** Inside the loop, for each block marked for ticking, the `process` method of the corresponding FluidTicker (e.g., DefaultFluidTicker) is executed.
> 6.  **State Mutation:** The FluidTicker logic directly modifies the chunk's FluidSection and BlockSection data, simulating flow, level changes, and new block creation.
> 7.  **Pipeline Completion:** Once the loop terminates, the modified chunk data is handed back to the engine's loading process, now containing a pre-settled fluid state.
>
> Final Chunk Data -> Sent to Player Client


---
description: Architectural reference for BlockTickPlugin
---

# BlockTickPlugin

**Package:** com.hypixel.hytale.builtin.blocktick
**Type:** Singleton

## Definition
```java
// Signature
public class BlockTickPlugin extends JavaPlugin implements IBlockTickProvider {
```

## Architecture & Concepts
The BlockTickPlugin is the central authority for the server's random block ticking system. This system is responsible for game mechanics like crop growth, ice melting, or leaf decay.

This plugin does not execute the block ticks itself. Instead, it acts as a configuration and initialization hub that injects the necessary logic into the core server engine. Its primary architectural function is to bridge static asset definitions with the dynamic game world. It reads block configuration from game assets to determine *which* blocks are capable of ticking and *how* they should behave, then registers this logic with the appropriate engine subsystems.

The core design pattern is an **initialization-time discovery optimization**. To avoid performance-intensive checks on every block during every game tick, the plugin pre-processes chunks as they are generated. It scans every block within a new chunk once, identifies those with ticking behavior defined in their assets, and sets a persistent flag on them. The downstream ticking systems, registered by this plugin, then operate only on this much smaller, pre-filtered set of flagged blocks, resulting in a significant performance gain.

It performs three critical setup tasks:
1.  **Codec Registration:** It registers JSON codecs for different types of `TickProcedure`, allowing content creators to define custom ticking logic in asset files.
2.  **System Injection:** It registers multiple systems (e.g., `ChunkBlockTickSystem`, `MergeWaitingBlocksSystem`) into the global `ChunkStore` registry. These systems contain the actual logic that is executed by the game loop to process ticked blocks.
3.  **Event Handling:** It listens for the `ChunkPreLoadProcessEvent` to trigger its discovery process on newly generated chunks.

## Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's plugin loading mechanism during the server bootstrap sequence. The constructor is invoked with a `JavaPluginInit` context, which is a standard part of the Hytale plugin lifecycle.
-   **Scope:** Application-level singleton. The plugin persists for the entire lifetime of the server process. A static `instance` field holds the single reference, which is set in the constructor and never modified.
-   **Destruction:** The object is destroyed only when the Java Virtual Machine for the server process terminates. It does not implement any explicit shutdown or resource cleanup logic.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its sole piece of internal state is the static `instance` reference, which is written once during initialization and subsequently treated as immutable. The plugin does not cache world data or maintain any session-specific state; its role is purely to configure other, stateful systems.
-   **Thread Safety:** The design is inherently thread-safe.
    -   The `setup` method is guaranteed to be called only once from the main server thread during startup.
    -   The `discoverTickingBlocks` method is called from the world-generation thread context. Its operations are confined to the specific chunk being processed, preventing race conditions with other world operations.
    -   The `getTickProcedure` method is a read-only operation that queries a globally managed and concurrent-safe `BlockTypeAssetMap`.

## API Surface
The public API is minimal, reflecting its role as a background configuration service rather than a tool for direct use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static BlockTickPlugin | O(1) | Retrieves the global singleton instance of the plugin. |
| getTickProcedure(int blockId) | TickProcedure | O(1) | Implements the IBlockTickProvider interface. Fetches the ticking logic for a given block ID from the asset map. |
| discoverTickingBlocks(Holder, WorldChunk) | int | O(N) | Scans all blocks in a chunk's sections, flags those with tick procedures, and returns the count of blocks found. |

## Integration Patterns

### Standard Usage
This plugin is not intended for direct interaction by other plugins or game code. Its functionality is exposed implicitly through the systems it registers. The primary integration point is the `BlockTickManager`, which this plugin configures as its provider.

Other engine systems that need to know how a block ticks will query the manager, not this plugin directly.

```java
// Engine code retrieving the tick procedure for a block
// Note: This does NOT reference BlockTickPlugin directly.

// 1. The BlockTickManager was configured by the plugin at startup
IBlockTickProvider provider = BlockTickManager.getBlockTickProvider();

// 2. The engine can now query the provider for any block ID
TickProcedure procedure = provider.getTickProcedure(someBlockId);

if (procedure != null) {
    // Execute the ticking logic
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BlockTickPlugin()`. The server's plugin loader is solely responsible for its creation. Use the static `BlockTickPlugin.get()` method if a reference is absolutely necessary.
-   **Manual Setup:** Do not invoke the `setup` method manually. Doing so will cause duplicate system registrations and event listeners, leading to severe and unpredictable behavior.
-   **Misuse of Discovery:** Do not call `discoverTickingBlocks` on chunks that have already been loaded and processed. This method is optimized for and intended to be used only on newly generated chunks. Calling it redundantly offers no benefit and may cause unnecessary disk writes by marking the chunk for saving.

## Data Pipeline
The plugin operates within two distinct data flows: a one-time setup flow and a recurring chunk generation flow.

**Flow 1: Server Initialization**
> Server Startup → Plugin Loader Instantiates `BlockTickPlugin` → **BlockTickPlugin.setup()** → Registers `ChunkBlockTickSystem` with `ChunkStore` → Registers `discoverTickingBlocks` with `EventRegistry` → Sets self as provider in `BlockTickManager`

**Flow 2: Chunk Generation & Ticking**
> World Generator Creates Chunk → Fires `ChunkPreLoadProcessEvent` → **BlockTickPlugin.discoverTickingBlocks()** → Scans Blocks & Sets Ticking Flag in `BlockSection` Data → Game Tick → `ChunkBlockTickSystem` Executes → Queries `BlockTickManager` → **BlockTickPlugin.getTickProcedure()** → System Executes Procedure → World State Changes


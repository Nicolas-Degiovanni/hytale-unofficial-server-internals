---
description: Architectural reference for GeneratedChunk
---

# GeneratedChunk

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class GeneratedChunk {
```

## Architecture & Concepts
The GeneratedChunk class is a critical data structure within the server's world generation pipeline. It functions as an **intermediate representation** of a world chunk, acting as a container for all data produced by a world generator *before* that data is integrated into the live, simulated world.

Its primary architectural role is to **decouple world generation algorithms from the live game state**. Generators operate on and populate a GeneratedChunk instance in isolation, without needing direct access to the main World object. This class aggregates the three fundamental components of a chunk:
*   Block data (via GeneratedBlockChunk)
*   Block state and component data (via GeneratedBlockStateChunk)
*   Entity data (via GeneratedEntityChunk)

The final assembly of this raw data into a game-ready WorldChunk is handled by the toWorldChunk method. This method represents the handoff point where abstract, generated data is transformed into a concrete, stateful object that can be managed by the server's core chunk systems.

## Lifecycle & Ownership
- **Creation:** A GeneratedChunk is instantiated by a world generation task, typically deep within a WorldGenerator or BiomeGenerator implementation. The generator is the sole owner of the instance during the population phase.
- **Scope:** The lifecycle of a GeneratedChunk is extremely short. It exists only for the duration of a single chunk generation operation. It is a transient object, not intended for long-term storage.
- **Destruction:** The object becomes eligible for garbage collection immediately after the toWorldChunk method is called and the resulting WorldChunk is passed to the World or ChunkManager. It holds no persistent references and is not registered in any global context.

## Internal State & Concurrency
- **State:** Effectively **Immutable** after construction. All internal fields are final, meaning the references to the underlying data containers (GeneratedBlockChunk, etc.) cannot be changed once the GeneratedChunk is instantiated. The data *within* those containers is mutated during the generation process, but the top-level structure is fixed.
- **Thread Safety:** **Not thread-safe**. This class is designed to be owned and operated on by a single world generation worker thread. It contains no internal locks or synchronization primitives.

    **WARNING:** Sharing a GeneratedChunk instance across multiple threads for concurrent modification will result in data corruption and race conditions. All population and finalization must occur on the owning thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toWorldChunk(World world) | Holder<ChunkStore> | O(N) | **Primary Method.** Assembles the contained data into a live WorldChunk, ready for integration into the game. Throws if world is null. |
| getBlockChunk() | GeneratedBlockChunk | O(1) | Retrieves the container for raw block data. |
| getBlockStateChunk() | GeneratedBlockStateChunk | O(1) | Retrieves the container for raw block state and component data. |
| getEntityChunk() | GeneratedEntityChunk | O(1) | Retrieves the container for raw entity data. |

## Integration Patterns

### Standard Usage
The canonical use case involves a world generator creating a new GeneratedChunk, populating its constituent parts, and then calling toWorldChunk to finalize the process.

```java
// Within a world generation task...
GeneratedChunk generatedChunk = new GeneratedChunk();

// The generator populates the internal data structures
// by operating on the results of the getters.
populateTerrain(generatedChunk.getBlockChunk());
placeFeatures(generatedChunk.getBlockStateChunk());
spawnCreatures(generatedChunk.getEntityChunk());

// Finalize and convert to a live world chunk
World targetWorld = context.getWorld();
Holder<ChunkStore> liveChunkHolder = generatedChunk.toWorldChunk(targetWorld);

// The live chunk is now ready to be loaded by the server
world.getChunkManager().scheduleLoad(liveChunkHolder);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not cache or store GeneratedChunk instances. They are transient and should be discarded after use to avoid memory leaks.
- **Instance Re-use:** Do not attempt to use the same GeneratedChunk instance to produce multiple WorldChunks. It is a single-use builder object.
- **Direct Modification Post-Generation:** Do not modify the internal state of a GeneratedChunk after it has been passed to another system or after toWorldChunk has been called.

## Data Pipeline
GeneratedChunk serves as a staging area in the data flow from abstract generation to concrete simulation.

> Flow:
> World Generator Algorithm -> Populates -> **GeneratedChunk** -> toWorldChunk() -> WorldChunk -> World ChunkManager


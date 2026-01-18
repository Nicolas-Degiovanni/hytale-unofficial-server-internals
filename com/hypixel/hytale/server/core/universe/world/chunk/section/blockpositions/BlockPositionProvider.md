---
description: Architectural reference for BlockPositionProvider
---

# BlockPositionProvider

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.blockpositions
**Type:** Component (Immutable Data Container)

## Definition
```java
// Signature
public class BlockPositionProvider implements Component<ChunkStore> {
```

## Architecture & Concepts
The BlockPositionProvider is a server-side component that functions as a performance-critical, read-only cache of block locations within a single BlockSection (a 16x16x16 volume). It is a core element of the engine's spatial query optimization strategy.

Its primary role is to eliminate the need for expensive, full-section block iteration. Instead of scanning up to 4096 blocks to find specific block types (e.g., all furnaces, all light sources), game systems can query this component to retrieve a pre-computed, filtered list. This pattern is essential for performance-sensitive systems like AI, which must frequently query the world state around them.

The component's data is considered a *snapshot* of the BlockSection at a specific point in time. The validity of this snapshot is managed through a change counter, ensuring that consumers do not operate on stale data after the underlying chunk has been modified. Due to its immutable nature, this component is inherently thread-safe and can be shared across multiple systems without synchronization.

### Lifecycle & Ownership
- **Creation:** A BlockPositionProvider is not instantiated directly by consumer systems. It is created by a dedicated world management system (e.g., a BlockPositionProviderSystem) that is responsible for scanning a BlockSection. When a section is modified, this system re-scans it, collects the relevant block positions, and constructs a *new* BlockPositionProvider instance. This new instance then replaces the old one on the corresponding ChunkStore entity.
- **Scope:** The lifetime of an instance is transient. It is valid only until the underlying BlockSection is modified. Once a change occurs, the instance is considered stale and is soon replaced and marked for garbage collection. It represents an immutable snapshot of a section's state.
- **Destruction:** An instance is implicitly destroyed and becomes eligible for garbage collection when it is replaced by a newer version on its parent ChunkStore component.

## Internal State & Concurrency
- **State:** Strictly immutable. The constructor guarantees that all internal collections are either cloned (BitSet) or wrapped in unmodifiable views (Int2ObjectMap). The public API further enforces this by returning clones of mutable structures. The clone method itself returns *this*, a clear indicator that the object's state cannot be changed after construction.
- **Thread Safety:** This component is unconditionally thread-safe. Its immutability allows any number of threads to read from it concurrently without locks or any other synchronization primitives. This is critical for server performance, as multiple entities and systems may query the same chunk data simultaneously.

## API Surface
The public contract is designed for efficient, read-only spatial queries.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isStale(blockSet, section) | boolean | O(1) | **CRITICAL:** Checks if the cached data is outdated. Compares internal change counters and searched block sets. Must be called before any query. |
| findBlocks(...) | void | O(N) | The primary query method. Populates a result list with block data within a specified range of an entity. N is the count of pre-filtered blocks, not the total blocks in the section. |
| getSearchedBlockSets() | BitSet | O(1) | Returns a clone of the BitSet indicating which block sets were indexed during its creation. |
| forEachBlockSet(...) | void | O(M) | Iterates over every indexed block set and its corresponding list of positions. M is the number of indexed block sets. |

## Integration Patterns

### Standard Usage
The correct usage pattern involves checking for staleness before querying. If the provider is stale, the consuming system should not use its data and may need to request a rebuild or fall back to a slower, direct lookup method.

```java
// Assume 'accessor' is a ComponentAccessor and 'chunkStoreRef' is a Ref<EntityStore>
// for the chunk entity.

BlockPositionProvider provider = accessor.getComponent(chunkStoreRef, BlockPositionProvider.getComponentType());
BlockSection section = world.getSectionAt(position); // The corresponding BlockSection

// CRITICAL: Always check for staleness before using the provider's data.
if (provider != null && !provider.isStale(TARGET_BLOCK_SET, section)) {
    List<IBlockPositionData> results = new ArrayList<>();
    provider.findBlocks(results, TARGET_BLOCK_SET, searchRange, yRange, entityRef, null, null, accessor);
    
    // Process the results...
} else {
    // The provider is stale or does not exist.
    // Either trigger a rebuild or use a slower fallback.
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring isStale:** The most severe anti-pattern is failing to call isStale before findBlocks. This will lead to logic bugs based on outdated world data, such as an AI failing to see a wall that was just placed, or trying to interact with a block that has been destroyed.
- **Direct Instantiation:** Never create an instance with *new BlockPositionProvider()*. The data it holds must be generated by the authoritative system that scans chunk sections to ensure data integrity and consistency with the world state.
- **Retaining References:** Do not hold a reference to a BlockPositionProvider across multiple game ticks. It is a transient snapshot. Always re-fetch it from the ChunkStore component each tick to ensure you have the latest version.

## Data Pipeline
The BlockPositionProvider acts as a cache in the middle of the world data flow. It has two distinct pipelines: one for population and one for consumption.

**Population Pipeline (Cache Write):**
> Flow:
> World Edit (Player/System places a block) -> BlockSection data is modified -> Change Counter is incremented -> **BlockPositionProviderSystem** detects stale section -> Scans all 4096 blocks -> Creates new **BlockPositionProvider** -> Replaces old component on ChunkStore

**Consumption Pipeline (Cache Read):**
> Flow:
> Game System (e.g., AI) -> Accesses ChunkStore -> Gets **BlockPositionProvider** -> Calls isStale() -> Calls findBlocks() -> Receives filtered List of block positions


---
description: Architectural reference for BlockPositionEntryGenerator
---

# BlockPositionEntryGenerator

**Package:** com.hypixel.hytale.server.npc.blackboard.view.blocktype
**Type:** Utility

## Definition
```java
// Signature
public class BlockPositionEntryGenerator {
```

## Architecture & Concepts
The BlockPositionEntryGenerator is a high-performance, specialized processor within the server-side NPC AI framework. Its primary function is to serve as a bridge between the raw world data representation, specifically a BlockChunk, and the AI's abstract world model, known as the Blackboard.

This class is responsible for executing targeted queries against a specific chunk section to find all blocks belonging to predefined categories, or BlockSets. It transforms the low-level block data into a structured, queryable BlockPositionProvider object, which is then consumed by NPC behaviors and decision-making systems.

The implementation heavily relies on the fastutil library for primitive collections, minimizing object allocation and garbage collection overhead. This design choice is critical for server performance, as environmental scanning is a frequent and potentially expensive operation.

A key internal feature is the use of reservoir sampling within its nested FoundBlockConsumer. When scanning for a common block type, this prevents the system from storing an excessive number of positions by maintaining a random, representative sample up to a configurable maximum. This is a crucial optimization for managing memory usage in dense environments.

## Lifecycle & Ownership
- **Creation:** BlockPositionEntryGenerator is a simple Plain Old Java Object (POJO). It is intended to be instantiated by a higher-level manager system responsible for populating the NPC blackboard, such as a per-world or per-region perception manager. It is not a singleton and does not manage its own lifecycle.

- **Scope:** An instance of this class can be long-lived and reused for multiple generation operations to reduce object churn. However, all internal state related to a scan is scoped exclusively to a single invocation of the *generate* method. The internal FoundBlockConsumer is explicitly initialized and released for each call.

- **Destruction:** The object is subject to standard Java garbage collection when it is no longer referenced by its owner. It holds no native resources and requires no explicit destruction.

## Internal State & Concurrency
- **State:** The class maintains a mutable internal state consisting of a reusable FoundBlockConsumer and an IntSet. This state is an optimization to avoid heap allocations on every scan. The state is reset at the beginning of each *generate* call and is not intended to persist between calls.

- **Thread Safety:** **This class is not thread-safe.** The mutable, reused internal state fields (foundBlockConsumer, internalIdHolder) make it inherently unsafe for concurrent access. If multiple threads call the *generate* method on the same instance simultaneously, it will result in state corruption and unpredictable behavior.

    **WARNING:** Each worker thread performing environmental scans must have its own dedicated instance of BlockPositionEntryGenerator, or access to a shared instance must be protected by external synchronization. The preferred pattern is to use an object pool or a ThreadLocal provider.

## API Surface
The public contract consists of a single method for executing the scan.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(changeCounter, sectionIndex, chunk, unifiedBlocksOfInterest, searchedBlockSets) | BlockPositionProvider | O(N) | Scans a chunk section for specified blocks. N is the number of blocks in the section. Returns a provider containing the found block positions, organized by BlockSet. |

## Integration Patterns

### Standard Usage
The generator should be invoked by a system managing AI perception updates. The caller is responsible for providing the correct chunk data and the set of blocks the AI is currently interested in.

```java
// Assumes an owning system provides the generator and context
BlockPositionEntryGenerator generator = perceptionManager.getGenerator();
BlockChunk targetChunk = world.getChunkAt(chunkPos);
IntList blocksToFind = aiBehavior.getRelevantBlockTypes();
BitSet setsToScan = aiBehavior.getRelevantBlockSets();

// Generate the data for the AI's blackboard
BlockPositionProvider results = generator.generate(
    targetChunk.getChangeCounter(),
    sectionIndex,
    targetChunk,
    blocksToFind,
    setsToScan
);

// Store the results in the AI's world model
npc.getBlackboard().updateBlockView(results);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Access:** Do not share a single instance of BlockPositionEntryGenerator across multiple threads without external locking. This will lead to race conditions and corrupted data.

- **Ignoring Stale Data:** The returned BlockPositionProvider is a snapshot in time. The *changeCounter* parameter is a critical part of the contract. Systems consuming this data must be aware that the underlying BlockChunk can be modified, rendering the results stale. Do not cache the BlockPositionProvider indefinitely without a mechanism to detect and refresh it.

## Data Pipeline
The generator processes data in a linear, single-pass flow. It efficiently filters a large volume of world data into a small, structured result set for AI consumption.

> Flow:
> AI Perception System Request -> **BlockPositionEntryGenerator.generate()** -> BlockSection.find() -> FoundBlockConsumer (with Reservoir Sampling) -> BlockPositionProvider -> AI Blackboard<ctrl63>


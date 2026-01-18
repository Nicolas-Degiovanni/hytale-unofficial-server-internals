---
description: Architectural reference for BlockPlacementMaskRegistry
---

# BlockPlacementMaskRegistry

**Package:** com.hypixel.hytale.server.worldgen.loader.prefab
**Type:** Transient

## Definition
```java
// Signature
public class BlockPlacementMaskRegistry extends FileMaskCache<BlockPlacementMask> {
```

## Architecture & Concepts
The BlockPlacementMaskRegistry is a specialized, high-performance caching and object-interning system used within the server's world generation pipeline. Its primary function is to dramatically reduce the memory footprint associated with prefab processing by ensuring that identical block placement rules are represented by a single, canonical object instance in memory.

This class operates as a second-level cache. It inherits from FileMaskCache, which provides a file-based caching layer, and adds a critical in-memory interning layer on top. During world generation, thousands of prefabs may be loaded, many of which could share identical placement masks. Without this registry, each instance would result in a new object allocation, leading to significant memory overhead. The registry canonicalizes these objects, replacing redundant instances with a single, shared reference.

This is achieved through a classic interning pattern:
1. A temporary, mutable "template" object is configured with the desired state.
2. The internal map is checked for an existing object that is equal to the template.
3. If a match is found, the existing, canonical instance is returned.
4. If no match is found, the template object is added to the map as the new canonical instance, and a new template is allocated for subsequent operations.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level world generation coordinator or prefab loading service. It is not a global singleton and should not be treated as one.
- **Scope:** The lifecycle of a BlockPlacementMaskRegistry is tightly coupled to a specific, bounded task, such as the generation of a world region or the loading of a zone's prefab set. Its internal state is only relevant for the duration of that task.
- **Destruction:** The registry and all canonical instances it holds are designed to be garbage collected once the parent world generation process completes. There is no explicit cleanup or shutdown method.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The *masks* and *entries* maps grow dynamically as the system encounters new, unique mask configurations during a world generation session. The registry is effectively a write-heavy, stateful cache.

- **Thread Safety:** **CRITICAL WARNING:** This class is **not thread-safe**. The internal use of HashMap and the "mutate-then-put" pattern on its temporary fields (tempMask, tempEntry) makes it fundamentally unsafe for concurrent access. Concurrent calls to its methods will lead to race conditions, resulting in data corruption within the internal maps and unpredictable world generation errors. All interactions with a BlockPlacementMaskRegistry instance **must** be confined to a single thread or be protected by an external synchronization mechanism.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| retainOrAllocateMask(defaultMask, specificMasks) | BlockPlacementMask | Amortized O(1) | Returns a shared, canonical BlockPlacementMask instance for the given configuration. |
| retainOrAllocateEntry(blocks, replace) | BlockPlacementMask.BlockArrayEntry | Amortized O(1) | Returns a shared, canonical BlockArrayEntry instance for the given block array. |

## Integration Patterns

### Standard Usage
The registry is intended to be used by prefab loaders to canonicalize mask data immediately after it has been parsed. This ensures memory efficiency before the mask is attached to the final prefab object model.

```java
// Assume 'loaderContext' provides access to the session-specific registry
BlockPlacementMaskRegistry registry = loaderContext.getMaskRegistry();

// Data parsed from a prefab file
IMask defaultMaskData = parseDefaultMask();
Long2ObjectMap<Mask> specificMasksData = parseSpecificMasks();

// Obtain the single, canonical instance of the mask
BlockPlacementMask canonicalMask = registry.retainOrAllocateMask(defaultMaskData, specificMasksData);

// Attach the canonical instance to the prefab
prefab.setPlacementMask(canonicalMask);
```

### Anti-Patterns (Do NOT do this)
- **Multi-threaded Access:** Never share a single BlockPlacementMaskRegistry instance across multiple threads without external locking. This will corrupt its internal state. If parallel prefab loading is required, each worker thread should be given its own separate registry instance.
- **Long-Term Caching:** Do not hold a reference to this registry beyond the scope of the world generation task it was created for. Doing so would constitute a memory leak, preventing the garbage collector from reclaiming the potentially large number of canonical mask objects it manages.

## Data Pipeline
The BlockPlacementMaskRegistry does not participate in a streaming data pipeline. Instead, it acts as a lookup and allocation service that sits between data parsing and object model construction.

> Flow:
> Prefab File Parser -> Raw Mask Data -> **BlockPlacementMaskRegistry** -> Canonical Mask Instance -> In-Memory Prefab Object


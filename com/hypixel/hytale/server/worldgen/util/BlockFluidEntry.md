---
description: Architectural reference for BlockFluidEntry
---

# BlockFluidEntry

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Value Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
@Deprecated
public record BlockFluidEntry(int blockId, int rotation, int fluidId) {
```

## Architecture & Concepts

**WARNING: This class is deprecated and scheduled for removal. It must not be used in new development. All usage should be migrated to the recommended alternative.**

BlockFluidEntry is an immutable data record designed to represent the state of a single voxel coordinate within the world generation pipeline. It serves as a simple, transient container that couples a solid block's identity and rotation with the identity of any fluid occupying the same space.

Architecturally, this record acts as a low-level data structure. Its purpose is to transfer composite voxel data between different stages of a world generation algorithm, such as from a biome-specific generator to a chunk serialization system. Its immutability, enforced by its declaration as a Java record, ensures that data remains consistent and free from side effects as it is passed between systems.

The existence of static constants like EMPTY and EMPTY_ARRAY indicates its intended use in performance-critical code paths, where avoiding unnecessary object allocations is paramount.

## Lifecycle & Ownership

As a simple value object, BlockFluidEntry has a trivial and transient lifecycle. It holds no resources and manages no external state.

-   **Creation:** Instantiated on-demand by world generation algorithms when a specific block and fluid combination needs to be represented. It is created, used, and discarded rapidly.
-   **Scope:** The scope of a BlockFluidEntry instance is almost always confined to a single method or a short-lived computational task. It is not designed for long-term storage.
-   **Destruction:** Instances are managed entirely by the Java Garbage Collector. They become eligible for collection as soon as they are no longer referenced, which is typically upon exiting the method in which they were created.

## Internal State & Concurrency

-   **State:** **Immutable**. By virtue of being a Java record, its state (blockId, rotation, fluidId) is established at the moment of instantiation and cannot be modified thereafter.
-   **Thread Safety:** **Fully thread-safe**. Its immutability guarantees that it can be safely passed between threads without any need for locks, synchronization, or other concurrency control mechanisms.

## API Surface

The public contract is defined by the record's components and its static constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| blockId() | int | O(1) | Accessor for the block's unique identifier. |
| rotation() | int | O(1) | Accessor for the block's rotation value. |
| fluidId() | int | O(1) | Accessor for the fluid's unique identifier. |
| EMPTY | BlockFluidEntry | O(1) | A static, shared instance representing an empty voxel. Use this to avoid allocation. |
| EMPTY_ARRAY | BlockFluidEntry[] | O(1) | A static, shared, zero-length array. Use this instead of creating a new empty array. |

## Integration Patterns

### Standard Usage

**WARNING: The only correct "standard usage" for this deprecated class is to refactor existing code to remove it.**

The original intended usage was as a return type for functions that determine the contents of a single block in the world.

```java
// Legacy example of how this class was used. DO NOT REPLICATE.
private BlockFluidEntry getBlockAt(int x, int y, int z) {
    // ... complex world generation logic ...
    if (isAir(x, y, z)) {
        return BlockFluidEntry.EMPTY;
    }
    return new BlockFluidEntry(STONE_ID, 0, WATER_ID);
}
```

### Anti-Patterns (Do NOT do this)

-   **New Implementations:** Do not use this class in any new code. It is deprecated and will be removed.
-   **Unnecessary Allocation:** In legacy code, avoid creating new instances where a static constant can be used. For example, do not write `new BlockFluidEntry(0, 0, 0)` when `BlockFluidEntry.EMPTY` is available.
-   **Long-Term Storage:** Do not store instances of this class in long-lived collections or data structures. It is designed for transient data transfer, not as a persistent storage format.

## Data Pipeline

BlockFluidEntry is not a processing component; it is the data itself. It represents a single quantum of information within the world generation data flow.

> Flow:
> Noise Generator -> Biome Placement Logic -> Voxel Material Function -> **BlockFluidEntry** (as output) -> Chunk Builder -> Final Chunk Data Structure


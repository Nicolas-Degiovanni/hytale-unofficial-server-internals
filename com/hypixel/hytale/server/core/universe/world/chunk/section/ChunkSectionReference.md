---
description: Architectural reference for ChunkSectionReference
---

# ChunkSectionReference

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ChunkSectionReference {
```

## Architecture & Concepts
The ChunkSectionReference is a lightweight, immutable data structure that acts as a contextual pointer. Its primary role is to bundle a specific BlockSection with its parent BlockChunk and its vertical index within that chunk.

This class is fundamental to any system that operates on vertical slices of the world. Instead of passing three separate parameters (the chunk, the section, and the index), engine systems can pass a single, cohesive ChunkSectionReference object. This simplifies method signatures and encapsulates the hierarchical relationship between a chunk and its constituent sections. It is heavily utilized by world generation, lighting, physics, and block update propagation systems to provide necessary context for their operations.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by higher-level world management or gameplay logic systems. There is no central factory; it is instantiated directly via its public constructor whenever a system needs to operate on a section with full chunk context.
- **Scope:** Short-lived and method-scoped. A ChunkSectionReference is typically created at the beginning of an operation, passed to relevant subsystems, and becomes eligible for garbage collection once the top-level operation completes.
- **Destruction:** Managed entirely by the Java garbage collector. There are no explicit cleanup methods. Holding references to these objects for extended periods is strongly discouraged as it may prevent the underlying chunk from being unloaded.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are private and set only once within the constructor. There are no setters or methods that modify internal state, making instances inherently stable after creation.
- **Thread Safety:** **Conditionally Thread-Safe**. The ChunkSectionReference object itself is immutable and can be safely shared across threads. However, the objects it points to, BlockChunk and BlockSection, are mutable and have their own complex concurrency models.

    **WARNING:** Safe concurrent access to a ChunkSectionReference does **not** guarantee safe concurrent access to the underlying chunk or section data. All operations on the referenced objects must adhere to the world's threading and locking protocols.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunk() | BlockChunk | O(1) | Returns the parent BlockChunk that contains the section. |
| getSection() | BlockSection | O(1) | Returns the specific BlockSection being referenced. |
| getSectionIndex() | int | O(1) | Returns the vertical index (Y-level) of the section within its parent chunk. |

## Integration Patterns

### Standard Usage
The canonical use case is to create a reference when iterating through a chunk's sections or when a specific section is targeted for an operation that requires broader chunk context.

```java
// A system that needs to process a specific section of a known chunk
void processWorldSection(BlockChunk targetChunk, int sectionY) {
    BlockSection targetSection = targetChunk.getSection(sectionY);

    // A null section indicates an empty, uninitialized part of the chunk
    if (targetSection == null) {
        return;
    }

    // Create the reference to pass to a subsystem
    ChunkSectionReference ref = new ChunkSectionReference(targetChunk, targetSection, sectionY);
    lightingEngine.recalculateSunlight(ref);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Caching:** Do not store ChunkSectionReference instances in a cache or as a long-term member of another object. The underlying BlockChunk may be unloaded by the world engine, rendering the reference stale and potentially causing memory leaks or access to invalid data.
- **Assuming Non-Null:** Do not create a reference with a null BlockSection. While the constructor permits this, any system consuming the reference will almost certainly throw a NullPointerException. The caller is responsible for validating that the section exists before creating the reference.

## Data Pipeline
This class does not process data; it *is* the data. It functions as a contextual packet created by one system and consumed by another.

> Flow:
> World Query (e.g., `world.getChunkAt(...)`) -> **ChunkSectionReference (Creation)** -> Consumed by Subsystem (e.g., PhysicsEngine, LightingEngine) -> Discarded


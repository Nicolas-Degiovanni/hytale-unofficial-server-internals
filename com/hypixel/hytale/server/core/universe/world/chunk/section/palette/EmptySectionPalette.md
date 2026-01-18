---
description: Architectural reference for EmptySectionPalette
---

# EmptySectionPalette

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.palette
**Type:** Singleton

## Definition
```java
// Signature
public class EmptySectionPalette implements ISectionPalette {
```

## Architecture & Concepts
The EmptySectionPalette is the foundational component in the chunk section storage system, implementing a memory-optimized State pattern. It represents a chunk section (a 32x32x32 volume of blocks) that is entirely filled with the default block, ID 0, which is typically air.

Its primary architectural role is to serve as a highly efficient, zero-cost placeholder. Instead of allocating an array to store 32,768 identical block IDs, the server can reference this single, static instance. This design dramatically reduces the memory footprint for worlds with large empty spaces, such as sky or underground caverns.

This class is the base state in a promotion-demotion lifecycle. When a non-zero block ID is written to a section represented by this palette, the palette signals that it must be **promoted** to a more complex type, such as a HalfByteSectionPalette, which can store varied data. This state transition is the core mechanism that allows chunk sections to dynamically adapt their memory usage based on their contents.

## Lifecycle & Ownership
- **Creation:** The single `INSTANCE` is a static final field, instantiated once by the JVM during class loading. It is not created by any engine component during the server's runtime.
- **Scope:** Application-wide. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The object is garbage collected when the server shuts down and its class loader is unloaded. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It holds no instance-level data and its methods always produce the same output for a given input. It is a pure representation of the "empty" state.
- **Thread Safety:** Inherently thread-safe. As a stateless singleton, the `INSTANCE` can be safely accessed and used by any number of threads simultaneously without requiring locks or other synchronization primitives.

## API Surface
The public contract is designed to enforce the state transition logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(index, id) | ISectionPalette.SetResult | O(1) | The primary state transition trigger. Returns UNCHANGED if id is 0; otherwise, returns REQUIRES_PROMOTE. |
| get(index) | int | O(1) | Always returns 0, reflecting that every block in the section is empty. |
| promote() | ISectionPalette | O(1) | Returns a new HalfByteSectionPalette instance. This is the mechanism for upgrading the section's storage. |
| demote() | ISectionPalette | N/A | Throws UnsupportedOperationException. This palette is the base state and cannot be demoted further. |
| serializeForPacket(buf) | void | O(1) | Does nothing. This is a critical network optimization, as no data is needed to represent an empty section. |

## Integration Patterns

### Standard Usage
The owning `ChunkSection` object manages the palette reference. When a block is modified, the section must check the result of the `set` operation and handle the promotion request.

```java
// Inside a ChunkSection or similar manager class
ISectionPalette currentPalette = EmptySectionPalette.INSTANCE;
int newBlockId = 42; // A non-empty block

// Attempt to set a block
ISectionPalette.SetResult result = currentPalette.set(0, newBlockId);

// The palette signals it can't handle this state
if (result == ISectionPalette.SetResult.REQUIRES_PROMOTE) {
    // Promote the palette to a more capable type
    currentPalette = currentPalette.promote();
    // Re-apply the operation on the new palette
    currentPalette.set(0, newBlockId);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring SetResult:** Failing to check the return value of `set` is a critical error. If you call `set` with a non-zero ID and do not handle `REQUIRES_PROMOTE`, the world state will not be updated, leading to silent data loss.
- **State Assumption:** Never assume a palette reference will remain an EmptySectionPalette after a write operation. Always re-evaluate the palette's state or be prepared to handle a new, promoted instance.
- **Attempting Demotion:** Calling `demote` on an EmptySectionPalette will crash the calling thread. The logic for demotion should only exist in higher-level palettes and must check if the target state is empty.

## Data Pipeline
The EmptySectionPalette acts as a gatekeeper in the data flow, either absorbing no-op changes or triggering a pipeline upgrade.

> **Block Write Flow:**
> `World::setBlock` -> `ChunkSection::setBlock` -> **EmptySectionPalette::set** -> Returns `REQUIRES_PROMOTE` -> `ChunkSection` replaces its palette reference with a `HalfByteSectionPalette`.

> **Network Serialization Flow:**
> `Chunk Packet Creation` -> `ChunkSection::serialize` -> **EmptySectionPalette::serializeForPacket** -> No bytes are written to the network buffer. The client infers the section is empty from the palette type ID.


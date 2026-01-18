---
description: Architectural reference for CollisionTracker
---

# CollisionTracker

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class CollisionTracker extends BlockTracker {
```

## Architecture & Concepts
The **CollisionTracker** is a high-performance, specialized data structure designed to manage the state of block collisions for a single entity during a discrete physics tick. It extends the base **BlockTracker**, which provides the core mechanism for tracking a set of unique block coordinates (x, y, z). **CollisionTracker** enhances this by associating each tracked block with two additional pieces of data: a **BlockData** object and a **BlockContactData** object.

Architecturally, this class serves as a temporary, mutable cache within the narrow phase of collision detection. It uses a structure-of-arrays (SoA) approach with three parallel arrays: one for coordinates (inherited from **BlockTracker**), one for block metadata (**blockData**), and one for contact details (**contactData**). This layout is optimized for cache-friendly iteration during collision resolution calculations.

The internal arrays have a fixed initial capacity and grow dynamically. This strategy amortizes the cost of memory allocation, avoiding performance penalties on every new tracked collision.

## Lifecycle & Ownership
- **Creation:** An instance of **CollisionTracker** is typically created on the stack or retrieved from an object pool at the beginning of an entity's physics update cycle. It is not a singleton and should not be shared globally.
- **Scope:** The object's lifetime is extremely short, intended to last for only a single physics simulation step. It accumulates all block contacts for an entity within that tick.
- **Destruction:** After the physics engine has used the tracked data to resolve collisions and apply impulses, the instance must be cleared via the **reset** method for reuse in the next tick. If not part of a pool, it is simply garbage collected.

**WARNING:** Failure to call **reset** between physics ticks will cause stale collision data to persist, leading to severe and difficult-to-debug physics anomalies.

## Internal State & Concurrency
- **State:** The **CollisionTracker** is fundamentally a mutable state container. Its entire purpose is to be populated, read, and cleared within a single frame or tick. The internal arrays hold references to mutable **BlockData** and **BlockContactData** objects.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. All internal operations, including array resizing (**alloc**), element removal (**untrack**), and state clearing (**reset**), perform direct array manipulations without any locking or synchronization. It is designed exclusively for use within a single, synchronized thread, such as the main server tick loop or a dedicated physics thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| track(x, y, z, contactData, blockData) | boolean | O(N) | Adds a new block collision if not already tracked. Returns true if the block was already present. |
| trackNew(x, y, z, contactData, blockData) | BlockContactData | O(1) amortized | Unconditionally adds a new block collision, resizing internal storage if necessary. Returns the stored contact data object. |
| untrack(index) | void | O(N) | Removes a tracked collision at a specific index, shifting subsequent elements to maintain a contiguous array. |
| getContactData(x, y, z) | BlockContactData | O(N) | Retrieves the contact data for a given block coordinate. Returns null if the block is not tracked. |
| reset() | void | O(N) | Clears all data for currently tracked blocks, preparing the instance for reuse. Does not deallocate internal arrays. |

## Integration Patterns

### Standard Usage
The **CollisionTracker** is designed to be used in a simple populate-resolve-reset cycle within a physics update.

```java
// Assume 'tracker' is a member field or retrieved from a pool
// In an entity's physics update method:

// 1. Prepare the tracker for the new tick
tracker.reset();

// 2. Populate with all detected collisions in the narrow phase
for (DetectedContact contact : narrowPhaseResults) {
    tracker.track(
        contact.blockX,
        contact.blockY,
        contact.blockZ,
        contact.getContactData(),
        contact.getBlockData()
    );
}

// 3. Use the tracked data for collision resolution
for (int i = 0; i < tracker.getCount(); i++) {
    BlockContactData contact = tracker.getContactData(i);
    // Apply physics response based on contact data...
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Sharing:** Do not share a single **CollisionTracker** instance across multiple entities simultaneously. Each entity's physics state must be tracked independently.
- **State Leakage:** Do not forget to call **reset** at the start of each physics tick. Failure to do so will cause collisions from the previous tick to affect the current one.
- **Concurrent Modification:** Do not read from a **CollisionTracker** on one thread while it is being written to on another. This will lead to data corruption, race conditions, and likely an **ArrayIndexOutOfBoundsException**.

## Data Pipeline
The **CollisionTracker** acts as a temporary state store within the server's physics pipeline. It collects and organizes data after the narrow phase of collision detection for consumption by the collision resolution stage.

> Flow:
> Physics Broad Phase -> Physics Narrow Phase -> **CollisionTracker.track()** -> Collision Resolution Logic -> Entity Velocity/Position Update


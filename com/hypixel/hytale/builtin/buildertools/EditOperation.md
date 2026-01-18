---
description: Architectural reference for EditOperation
---

# EditOperation

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class EditOperation {
```

## Architecture & Concepts
The EditOperation class is a stateful, short-lived object that represents a single, atomic world modification. It embodies the **Command Pattern**, encapsulating all information required to execute a change, and more importantly, to revert it. This class is the cornerstone of the builder tools' undo and redo functionality.

Its core design revolves around two internal BlockSelection objects: *before* and *after*.
-   The **before** selection acts as a backup, capturing the original state of any block *just before* it is modified by the operation. This is populated lazily using a capture-on-write strategy.
-   The **after** selection records the desired final state of all blocks affected by the operation.

To optimize performance for localized edits, an EditOperation does not interact directly with the global World state. Instead, it is initialized with a LocalCachedChunkAccessor. This provides a fast, localized view of the world, minimizing performance overhead during complex geometric calculations common in builder tools.

This class does not directly apply changes to the world. It serves as a data container or a "change-set" that is constructed by a tool and then passed to a higher-level system, such as an EditSessionManager, for final application and history tracking.

## Lifecycle & Ownership
-   **Creation:** An EditOperation is instantiated by a specific builder tool (e.g., a brush, fill, or shape tool) at the beginning of a user-initiated action. It requires a valid World context and bounding coordinates to define its scope.

-   **Scope:** The object's lifetime is tied to a single, discrete user action. For example, it may be created on a mouse-down event, populated with changes during a mouse-drag, and finalized on a mouse-up event.

-   **Destruction:** After the user action is complete, the populated EditOperation object is typically handed off to a session or history management service. This service may apply the changes and then push the EditOperation onto an undo stack. The object is eligible for garbage collection once it is removed from the undo stack or the session is terminated.

## Internal State & Concurrency
-   **State:** The internal state of an EditOperation is highly **mutable**. Its primary purpose is to accumulate block changes within its *before* and *after* BlockSelection fields. The state is built progressively through calls to methods like setBlock.

-   **Thread Safety:** This class is **not thread-safe**. It is designed for synchronous, single-threaded access within the scope of a single game tick or user input event. All interactions, from creation to finalization, must be confined to the main server thread. Concurrent modification from multiple threads will lead to a corrupted and unpredictable change-set.

## API Surface
The public API is designed for populating the operation with block and fluid changes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBlock(x, y, z, blockId, rotation) | boolean | O(1) amortized | Records a block change. Captures the original block state in the *before* selection if not already present, and adds the new state to the *after* selection. Returns false if the operation is blocked by the BlockMask or if the target chunk is not in memory. |
| setMaterial(x, y, z, material) | boolean | O(1) amortized | A convenience method that delegates to either setBlock or setFluid based on the provided Material type. |
| getBlock(x, y, z) | int | O(1) | Reads the current block ID from the underlying cached chunk accessor. Does not reflect changes made within the current operation. |
| getBefore() | BlockSelection | O(1) | Returns a reference to the selection containing the original state of all modified blocks. **Warning:** Direct modification is not recommended. |
| getAfter() | BlockSelection | O(1) | Returns a reference to the selection containing the desired final state of all modified blocks. **Warning:** Direct modification is not recommended. |
| getAccessor() | OverridableChunkAccessor | O(1) | Provides access to the underlying cached world view for this operation. |

## Integration Patterns

### Standard Usage
An EditOperation is typically used within a loop where a builder tool calculates the positions for a shape. The tool populates the operation, which is then passed to another system for execution.

```java
// A builder tool creates and populates the operation
EditOperation op = new EditOperation(world, x, y, z, range, min, max, mask);

for (Vector3i pos : shape.getPositions()) {
    op.setBlock(pos.getX(), pos.getY(), pos.getZ(), myBlockId);
}

// The populated operation is handed off to be applied and tracked
editSession.execute(op);
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not hold references to an EditOperation for extended periods. Its LocalCachedChunkAccessor may prevent chunks from being unloaded, potentially causing memory leaks.
-   **Concurrent Access:** Never share or modify an EditOperation instance across multiple threads. This will corrupt the *before* and *after* state, making undo operations unreliable.
-   **Direct State Modification:** Avoid calling methods on the returned *before* and *after* BlockSelection objects directly. Use the provided setBlock and setMaterial methods on EditOperation to ensure the capture-on-write logic is correctly executed.

## Data Pipeline
The EditOperation serves as a critical data structure that translates user intent into a reversible world change.

> Flow:
> User Input -> Builder Tool Logic -> **EditOperation (Population)** -> EditSessionManager -> World State Update & Undo Stack

---


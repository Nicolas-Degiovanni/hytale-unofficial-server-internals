---
description: Architectural reference for ClipboardBoundsSnapshot
---

# ClipboardBoundsSnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Value Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ClipboardBoundsSnapshot implements ClipboardSnapshot<ClipboardBoundsSnapshot> {
```

## Architecture & Concepts
The ClipboardBoundsSnapshot is an immutable data structure that captures the state of a player's three-dimensional selection area at a specific moment in time. It functions as a Memento within the Builder Tools plugin, providing the mechanism to save and restore the minimum and maximum corner vectors of a BlockSelection.

Its primary role is to facilitate state management for complex, multi-step building operations, most notably for implementing undo and redo functionality. By creating a snapshot before an operation that modifies the selection area (such as pasting or moving), the system can later revert the selection to its exact prior state by applying the snapshot. This class exclusively handles the geometric bounds and does not concern itself with the block or entity data *within* those bounds.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand whenever the system needs to record the current selection bounds. This is typically triggered by a user action, such as copying a selection or executing a command that will alter the selection. The `restoreClipboard` method also creates a new instance to capture the state *before* the restoration, enabling the restoration itself to be undone.
- **Scope:** Transient and short-lived. A snapshot's lifetime is tied to the operation it supports. For example, it may be pushed onto an undo stack within the BuilderState, where it persists until it is used for a restore operation or the undo history is cleared.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection as soon as it is no longer referenced, for instance, after being popped from an undo stack.

## Internal State & Concurrency
- **State:** Immutable. The internal `min` and `max` Vector3i fields are final and are assigned only during construction. This guarantees that a snapshot represents a consistent and unchangeable record of a past state. The static EMPTY instance provides a canonical representation for a null or zero-volume selection.
- **Thread Safety:** Inherently thread-safe due to its immutability. No synchronization mechanisms are required. An instance of ClipboardBoundsSnapshot can be safely shared across threads without risk of data corruption. However, its usage is typically confined to the main server thread that processes player commands and world updates.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ClipboardBoundsSnapshot(BlockSelection) | Constructor | O(1) | Creates a snapshot from an existing BlockSelection object. |
| ClipboardBoundsSnapshot(Vector3i, Vector3i) | Constructor | O(1) | Creates a snapshot from explicit minimum and maximum vectors. |
| getMin() | Vector3i | O(1) | Returns the minimum corner of the captured selection area. |
| getMax() | Vector3i | O(1) | Returns the maximum corner of the captured selection area. |
| restoreClipboard(...) | ClipboardBoundsSnapshot | O(1) + Network Latency | **Critical Operation.** Applies this snapshot's bounds to the player's current selection state. It returns a *new* snapshot of the state as it was *before* this restoration. |

**WARNING:** The `restoreClipboard` method has significant side effects. It directly mutates the `BuilderState` object and triggers a network packet to be sent to the client to update the visual selection box.

## Integration Patterns

### Standard Usage
This class is designed to be used as part of a command or state management pattern, such as an undo/redo stack. The correct pattern involves capturing the state, performing an action, and then using the captured state to revert the action if needed.

```java
// Assumes 'builderState' is the current state for a player
// 1. Capture the current selection bounds before an operation
ClipboardBoundsSnapshot preOperationSnapshot = new ClipboardBoundsSnapshot(builderState.getSelection());

// 2. Perform an operation that modifies the selection
builderState.getSelection().setSelectionArea(newPos1, newPos2);
builderState.sendArea(); // Update client

// 3. To undo, use the snapshot. The return value is the snapshot of the state *before* the undo.
ClipboardBoundsSnapshot postOperationSnapshot = preOperationSnapshot.restoreClipboard(ref, player, world, builderState, accessor);

// 'postOperationSnapshot' can now be pushed to a redo stack.
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Return Value:** The `restoreClipboard` method returns a new snapshot of the state it just overwrote. Discarding this return value breaks the chain of state history and makes it impossible to implement a "redo" feature for the restore operation.
- **State Modification:** Do not attempt to modify the Vector3i objects returned by `getMin` or `getMax`. While Vector3i itself may be mutable in some contexts, the snapshot pattern relies on the immutability of the captured data. Modifying them breaks the integrity of the snapshot.
- **Misuse for Block Data:** This class only stores the coordinates of the selection box. It does not store any information about the blocks or entities contained within those bounds. Using it with the expectation of restoring world content will lead to incorrect behavior.

## Data Pipeline
ClipboardBoundsSnapshot is not part of a streaming data pipeline but rather a key component in a state-based control flow, typically initiated by a player command.

> Flow:
> Player Command (e.g., /undo) -> Command Handler -> **ClipboardBoundsSnapshot.restoreClipboard()** -> BuilderState Mutation -> Network Packet Sent to Client -> Client-side Visual Update

The object itself acts as a data payload, carrying state information from the point of capture to the point of restoration.


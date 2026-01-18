---
description: Architectural reference for SelectionSnapshot
---

# SelectionSnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface SelectionSnapshot<T extends SelectionSnapshot<?>> {
```

## Architecture & Concepts
The SelectionSnapshot interface defines a contract for capturing and restoring a stateful selection within the game world. It is a core component of the server-side Builder Tools system, forming the foundation for features like copy/paste, undo/redo, and prefab instantiation.

Architecturally, this interface embodies the **Memento Pattern**. An object implementing SelectionSnapshot acts as a *memento*—an opaque token that holds the state of a selection (e.g., blocks, entities, component data) at a specific moment in time. The system that created the snapshot (the *originator*) can later use this memento to restore the selection to its former state via the `restore` method.

The recursive generic type parameter, `T extends SelectionSnapshot<?>`, is a critical design choice. It ensures that the `restore` method returns a snapshot of the same or a compatible type, which is essential for creating a logical chain of operations, such as an undo history.

## Lifecycle & Ownership
As an interface, SelectionSnapshot itself has no lifecycle. However, objects that implement this contract have a well-defined and typically transient lifecycle.

- **Creation:** A concrete snapshot instance is created by a higher-level manager system, such as a `ClipboardManager` or `HistoryManager`, in response to a player action (e.g., executing a "copy" command). The snapshot encapsulates the state of the world at that exact moment.
- **Scope:** The lifetime of a snapshot object is determined by its container. For a clipboard, it persists until overwritten or the player session ends. For an undo/redo stack, it may persist for a limited number of operations before being discarded.
- **Destruction:** Instances are eligible for garbage collection once they are no longer referenced by any system. For example, when an undo stack is cleared, a player logs out, or a new selection is copied to the clipboard.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Concrete implementations are inherently **stateful and should be treated as immutable** after creation. They contain a complete representation of the captured selection. Modifying a snapshot's internal data post-creation is an anti-pattern that leads to unpredictable behavior.
- **Thread Safety:** Implementations are **not thread-safe**. The `restore` method performs direct, wide-ranging mutations on the core world state (World, EntityStore). Invoking this method from any thread other than the main server world-tick thread will result in severe data corruption, race conditions, and server instability. All interactions with this interface and its implementations must be synchronized with the main game loop.

## API Surface
The public contract consists of a single, powerful method for state restoration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| restore(Ref, Player, World, ComponentAccessor) | T | O(N) | Applies the snapshot to the world. N is the volume of the snapshot. Returns a new snapshot of the state *before* this restoration, enabling undo functionality. May return null if an undo snapshot cannot be generated. |

## Integration Patterns

### Standard Usage
The intended pattern involves a managing service that orchestrates the creation and application of snapshots. The client of this interface should never be concerned with the internal data of the snapshot, only with storing it and invoking the restore operation.

```java
// A manager system (e.g., a ClipboardManager) holds the snapshot
SelectionSnapshot currentClipboard = player.getClipboard().getSnapshot();

// When the player initiates a "paste" action, the manager restores it
if (currentClipboard != null) {
    // The restore operation returns a snapshot of the area *before* the paste
    // This is immediately pushed to the undo stack.
    SelectionSnapshot undoSnapshot = currentClipboard.restore(entityStoreRef, player, world, accessor);
    player.getHistoryManager().pushUndo(undoSnapshot);
}
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Restoration:** Never call `restore` from a separate thread or asynchronous task. World modification must be strictly confined to the server's main thread.
- **State Mutation:** Do not attempt to cast a snapshot to its concrete type to modify its internal data after it has been created. This breaks the memento pattern and will corrupt the state.
- **Long-Term Storage:** Holding onto large snapshots for extended periods (e.g., in a global cache) can lead to significant memory pressure. Snapshots are designed for transient, operation-specific use.

## Data Pipeline
SelectionSnapshot acts as a command object in a user-driven data flow. It does not process a stream of data but rather applies a static, self-contained dataset to the world state.

> Flow:
> Player Input (Copy Command) -> Builder Tool System -> **Creates ConcreteSelectionSnapshot** (captures World data) -> Stored in a Manager (e.g., Clipboard)
>
> Player Input (Paste Command) -> Manager retrieves Snapshot -> Builder Tool System -> **Calls restore() on Snapshot** -> World state is mutated


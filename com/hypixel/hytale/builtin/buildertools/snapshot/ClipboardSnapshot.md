---
description: Architectural reference for ClipboardSnapshot
---

# ClipboardSnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface ClipboardSnapshot<T extends SelectionSnapshot<?>> extends SelectionSnapshot<T> {
```

## Architecture & Concepts
The ClipboardSnapshot interface defines a contract for objects that represent a player-specific, in-memory copy of a region of the game world. It is a specialized form of a SelectionSnapshot, tailored specifically for the "copy/paste" functionality within the server's Builder Tools plugin.

Unlike a generic SelectionSnapshot which might be used for templating or administrative backups, a ClipboardSnapshot is intrinsically linked to a player's session and their current builder state. Its primary role is to hold serialized world data (blocks, entities, etc.) and provide the logic to restore this data into the world at a new location, effectively performing a "paste" operation.

The interface's default implementation of the `restore` method acts as a critical adapter. It translates a generic restore request into a clipboard-specific one by retrieving the player's unique BuilderState, which is required by the core `restoreClipboard` method. This design ensures that any paste operation is always executed within the context of the correct player's clipboard.

## Lifecycle & Ownership
As an interface, ClipboardSnapshot itself has no lifecycle. The lifecycle described here applies to concrete classes that implement this contract.

-   **Creation:** An object implementing ClipboardSnapshot is instantiated when a player executes a "copy" or "cut" command using the builder tools. The system captures a region of the world and serializes it into this new snapshot object. This object is then stored within the player's associated BuilderState.
-   **Scope:** The snapshot instance persists as long as it remains the active clipboard for a player. It is typically held as a field within the player's BuilderState object, which is tied to their server session. Its lifetime is therefore scoped to the player's session or until a new copy/cut operation replaces it.
-   **Destruction:** The object is eligible for garbage collection when the player's BuilderState is cleared or replaced. This occurs when the player logs out, performs a new copy/cut operation, or explicitly clears their clipboard.

## Internal State & Concurrency
-   **State:** Any class implementing this interface is inherently **stateful and mutable**. It contains a complete, serialized representation of a world region. This data is captured at a single point in time and does not update if the original world area changes.
-   **Thread Safety:** Implementations are **not thread-safe**. All methods, particularly `restoreClipboard`, perform direct, high-volume modifications to core server components like World and EntityStore. These operations must be executed exclusively on the main server thread to prevent catastrophic race conditions, data corruption, and world state desynchronization.

**WARNING:** Calling `restoreClipboard` from any thread other than the primary world tick thread will lead to undefined behavior and likely server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| restoreClipboard(...) | T | O(N) | Restores the snapshot into the world. N is the number of blocks/entities in the snapshot. Requires a valid player and BuilderState context. |
| restore(...) | T | O(N) | Default adapter method. Retrieves the player's BuilderState and delegates to `restoreClipboard`. Will fail if the player context is invalid. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve the current snapshot from a player's state and then invoke the restore method, providing the necessary world and entity context. This is typically done in response to a "paste" command from the player.

```java
// Assume 'player' and 'world' context is available
BuilderToolsPlugin.BuilderState state = BuilderToolsPlugin.getState(player);

if (state != null && state.getClipboard() != null) {
    ClipboardSnapshot snapshot = state.getClipboard();
    
    // The restore method handles finding the correct state and context
    snapshot.restore(entityStoreRef, player, world, componentAccessor);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Caching:** Do not retrieve a ClipboardSnapshot instance and hold a reference to it for an extended period. The player may perform another copy action, making your cached reference stale and obsolete. Always fetch the current clipboard from the BuilderState just before use.
-   **Cross-Player Usage:** Never attempt to use one player's ClipboardSnapshot in the context of another player. The restore logic is tied to the state and permissions of the original player who created the snapshot.
-   **Modification After Creation:** Do not attempt to modify the internal data of a snapshot object after it has been created. It is designed to be a point-in-time capture.

## Data Pipeline
The ClipboardSnapshot serves as a data container in a command-driven pipeline for world editing.

> Flow:
> Player "Copy" Command -> World Region Queried -> **New ClipboardSnapshot Instance (Data Serialized)** -> Stored in Player's BuilderState -> Player "Paste" Command -> **ClipboardSnapshot::restoreClipboard Invoked** -> Data Deserialized -> World & EntityStore Updated


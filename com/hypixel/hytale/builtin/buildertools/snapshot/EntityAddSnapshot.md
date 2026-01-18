---
description: Architectural reference for EntityAddSnapshot
---

# EntityAddSnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Transient

## Definition
```java
// Signature
public class EntityAddSnapshot implements EntitySnapshot<EntityRemoveSnapshot> {
```

## Architecture & Concepts
The EntityAddSnapshot is a command object, not a data-transfer object. It does not store the state of an entity. Instead, it represents the *event* of an entity being added to the world. This class is a fundamental component of the builder tools' undo/redo system, implementing a variation of the Command Pattern.

Its primary role is to provide the logic for reversing an entity creation action. When a player adds an entity using a builder tool, an EntityAddSnapshot is created and pushed onto an undo stack. The snapshot holds a direct, non-owning reference (a Ref) to the newly created entity.

The core architectural concept is that the `restoreEntity` method performs the *inverse* operation. For a snapshot representing an *addition*, the inverse is a *removal*. This design allows a generic undo manager to process a stack of EntitySnapshot objects without needing to know the specific type of each one; it simply calls the `restoreEntity` method to revert the game state.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level manager, such as a `SnapshotManager` or a specific `BuilderTool`, immediately after a new entity has been successfully added to the world. It captures the reference of the newly spawned entity.
-   **Scope:** The object's lifetime is tied to the undo/redo history of a player's session. It is typically stored in a collection, such as a `Deque` or `Stack`, representing the undo buffer. It persists only as long as the action it represents is available to be undone.
-   **Destruction:** The object is eligible for garbage collection once it is removed from the undo stack. This occurs when a player's undo history is cleared, a new action invalidates the old history, or the player's session ends.

## Internal State & Concurrency
-   **State:** Immutable. The internal `entityRef` is final and assigned at construction. The snapshot itself cannot be modified after creation. However, the underlying entity it points to is highly mutable and may be destroyed by other game systems. The `isValid()` check on the reference is therefore critical before any operation.
-   **Thread Safety:** This class is immutable and thus inherently thread-safe. **WARNING:** The methods it exposes, particularly `restoreEntity`, operate on core world state (World, EntityStore). These systems are **not** thread-safe. All interactions with an EntityAddSnapshot that modify the world must be executed exclusively on the main server thread to prevent race conditions and world state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| restoreEntity(player, world, accessor) | EntityRemoveSnapshot | O(1) | Reverts the entity addition by removing the entity from the world. This is the "undo" operation. It returns the inverse command, an EntityRemoveSnapshot, which can be used by a "redo" system. Returns null if the entity reference is no longer valid (e.g., the entity was already removed by another system). |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most systems. It is designed to be managed by an undo/redo service. The service would pop this snapshot from its undo stack and invoke `restoreEntity` to revert the world state.

```java
// Hypothetical UndoManager logic
EntitySnapshot<?> lastAction = undoStack.pop();

// The restoreEntity call performs the actual world modification
EntitySnapshot<?> redoAction = lastAction.restoreEntity(player, world, accessor);

if (redoAction != null) {
    redoStack.push(redoAction);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Caching:** Do not hold a reference to an EntityAddSnapshot for longer than its lifetime in the undo/redo stack. The underlying entity it refers to can be destroyed at any time, making the snapshot's reference invalid.
-   **Cross-Thread Modification:** Never call `restoreEntity` from an asynchronous task or any thread other than the main server game loop thread. This will lead to severe and difficult-to-debug concurrency issues.
-   **Manual Reversal:** Do not manually remove the entity from the world and then call `restoreEntity`. The method is designed to be the single source of truth for the reversal operation and handles the necessary state checks.

## Data Pipeline
EntityAddSnapshot acts as a command within a state-change pipeline, rather than a data-processing pipeline. Its flow is triggered by user input and managed by the undo system.

> Flow:
> User places an entity -> `World.addEntity` -> **EntityAddSnapshot created** with new entity Ref -> Snapshot pushed to Undo Stack -> User triggers "Undo" -> Snapshot popped from stack -> **`restoreEntity` called** -> `EntityStore.removeEntity` -> World state is reverted.


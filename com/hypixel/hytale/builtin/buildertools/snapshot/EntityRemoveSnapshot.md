---
description: Architectural reference for EntityRemoveSnapshot
---

# EntityRemoveSnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Transient

## Definition
```java
// Signature
public class EntityRemoveSnapshot implements EntitySnapshot<EntityAddSnapshot> {
```

## Architecture & Concepts
The EntityRemoveSnapshot is a Command pattern object that functions as a memento for a deleted entity. It is a core component of the server-side builder tools undo/redo system.

When an entity is removed from the world via a tracked builder tool action, an instance of this class is created. It captures a complete, point-in-time copy of the entity's component data. This snapshot is self-contained and holds all information necessary to perfectly reconstruct the entity at a later time.

Its primary role is to provide the `restoreEntity` operation, which reverses the deletion by re-injecting the captured entity data back into the world's Entity-Component System (ECS). The class acts as a bridge between a high-level user action (like "undo deletion") and the low-level ECS manipulation required to bring an entity back into existence.

## Lifecycle & Ownership
- **Creation:** Instantiated by a command-handling service or a specific builder tool immediately before an entity is removed from the world. The constructor requires a `Ref<EntityStore>` to the live entity that is about to be deleted.
- **Scope:** The object's lifetime is bound to the undo history of a player's session. It is typically stored within a `Stack` or `Deque` data structure managed by a session-specific UndoManager. It holds no external resources and is lightweight.
- **Destruction:** The object is eligible for garbage collection when it is popped from the undo stack without being used, or when the entire undo history is cleared. This typically occurs when a player's session ends or the undo history limit is reached.

## Internal State & Concurrency
- **State:** Immutable. The internal `Holder<EntityStore>` is a deep copy of the entity's component data, created during construction. Once an EntityRemoveSnapshot is created, its state cannot be altered. It represents a historical fact about the game world and does not track subsequent changes.
- **Thread Safety:** The object's state is immutable, making the instance itself inherently thread-safe and suitable for being passed between threads. However, methods that interact with the world state, specifically `restoreEntity`, are **not** thread-safe.

    **WARNING:** The `restoreEntity` method modifies the world's ECS state via the `ComponentAccessor`. It **must** be invoked exclusively from the main server thread (the primary game loop) to prevent catastrophic race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| restoreEntity(player, world, accessor) | EntityAddSnapshot | O(C) | Re-creates the entity in the specified world using the snapshot's data. C is the number of components on the entity. Returns the inverse command, an EntityAddSnapshot, for the redo stack. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. It is managed by a higher-level undo/redo service. The typical flow involves creating the snapshot, performing the deletion, and pushing the snapshot onto an undo stack.

```java
// Conceptual example within an UndoManager
// 1. An entity is about to be deleted by a tool
Ref<EntityStore> entityToDelete = world.findEntity(...);

// 2. Create a snapshot to remember its state
EntityRemoveSnapshot undoCommand = new EntityRemoveSnapshot(entityToDelete);
undoStack.push(undoCommand);

// 3. Now, perform the actual deletion
world.removeEntity(entityToDelete);

// 4. Later, to perform an "undo"
if (!undoStack.isEmpty()) {
    EntityRemoveSnapshot command = undoStack.pop();
    ComponentAccessor<EntityStore> accessor = world.getEntities();

    // Restore the entity and get the "redo" command
    EntityAddSnapshot redoCommand = command.restoreEntity(player, world, accessor);
    redoStack.push(redoCommand);
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not hold a reference to an EntityRemoveSnapshot for longer than its lifetime in the undo/redo stack. It represents historical state and should not be used for any other purpose.
- **Asynchronous Restoration:** Never call `restoreEntity` from a separate thread or asynchronous task. All world modifications must be synchronized with the main server tick. Failure to do so will lead to unpredictable and severe server errors.

## Data Pipeline
The EntityRemoveSnapshot is a key element in the data flow for reversible entity operations. It captures data from the live game state and enables its re-injection later.

> **Deletion Flow:**
> Player Action (e.g., Delete Tool) → Server Command Handler → Get `Ref<EntityStore>` for target entity → **EntityRemoveSnapshot(ref)** → Store snapshot in Undo Stack → Remove entity from `World`.

> **Restoration (Undo) Flow:**
> Player Action (Undo) → Pop **EntityRemoveSnapshot** from Undo Stack → Call `restoreEntity` → `ComponentAccessor` adds entity data to `World` → `EntityAddSnapshot` is returned → Store new snapshot in Redo Stack.


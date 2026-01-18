---
description: Architectural reference for EntityTransformSnapshot
---

# EntityTransformSnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Transient Value Object

## Definition
```java
// Signature
public class EntityTransformSnapshot implements EntitySnapshot<EntityTransformSnapshot> {
```

## Architecture & Concepts
The EntityTransformSnapshot is a server-side data structure that implements the Memento design pattern. Its sole purpose is to capture and store the spatial state of an entity—specifically its world transform and head rotation—at a single moment in time. This class is a fundamental building block for systems requiring state rollback, such as the undo/redo functionality within the Hytale Builder Tools.

It functions as an immutable, point-in-time record. Upon instantiation, it performs a deep copy of the relevant component data, ensuring that the snapshot is completely decoupled from the live entity. Subsequent changes to the entity in the game world will not affect the data stored within an existing snapshot instance.

This class operates exclusively within the server's Entity Component System (ECS). It safely interacts with entity data through the standard ComponentAccessor, which is the designated abstraction for reading and writing component state.

### Lifecycle & Ownership
- **Creation:** An EntityTransformSnapshot is instantiated on-demand by a higher-level system, typically a command handler or tool logic, immediately before an operation modifies an entity's transform. This captures the "before" state required for a potential undo action.
- **Scope:** The object is short-lived and has no persistent identity. Its lifetime is bound to the system that created it, most commonly an undo stack (e.g., a `java.util.Stack`). It exists only as long as the operation it represents is reversible.
- **Destruction:** The object is managed by the Java garbage collector. It is reclaimed once all references to it are dropped, for example, when an undo stack is cleared or an entry is popped and no longer needed. It has no explicit `destroy` or `close` method.

## Internal State & Concurrency
- **State:** **Immutable**. The internal fields `ref`, `transform`, and `headRotation` are set once in the constructor and are never modified thereafter. The constructor explicitly clones the transform and rotation vector data, guaranteeing that the snapshot's state cannot be mutated by external changes to the original entity's components.

- **Thread Safety:** **Conditionally Safe**. As an immutable object, an instance of EntityTransformSnapshot can be safely read by multiple threads. However, its primary method, `restoreEntity`, modifies the state of the game world.

    **WARNING:** The `restoreEntity` method is **not thread-safe** and **must** be invoked exclusively from the main server thread which governs the game loop and has write-access to the ECS. Calling this method from any other thread will lead to race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| restoreEntity(player, world, accessor) | EntityTransformSnapshot | O(1) | Applies the snapshot's stored transform and head rotation to the target entity. This method is the primary mechanism for state restoration. It returns a new snapshot of the entity's state *before* the restoration, which can be used for a "redo" operation. Returns null if the entity reference is no longer valid (i.e., the entity has been deleted). |

## Integration Patterns

### Standard Usage
The class is designed to be used in a classic undo/redo stack pattern. A snapshot is created and pushed onto an "undo" stack before an action is performed. To undo the action, the snapshot is popped and its `restoreEntity` method is called.

```java
// Context: A command is about to move an entity
ComponentAccessor accessor = world.getComponentAccessor(EntityStore.class);
Ref<EntityStore> entityToModify = ...;

// 1. Capture the "before" state and push to the undo stack
EntityTransformSnapshot undoState = new EntityTransformSnapshot(entityToModify, accessor);
playerUndoHistory.getUndoStack().push(undoState);

// 2. Perform the entity modification
TransformComponent tc = accessor.getComponent(entityToModify, TransformComponent.getComponentType());
tc.setPosition(newPosition);

// ... Later, when the player triggers an undo ...

// 3. Pop the last state and restore it
EntityTransformSnapshot snapshotToRestore = playerUndoHistory.getUndoStack().pop();
if (snapshotToRestore != null) {
    // 4. The restore operation returns the "before-restore" state, perfect for a redo stack
    EntityTransformSnapshot redoState = snapshotToRestore.restoreEntity(player, world, accessor);
    if (redoState != null) {
        playerUndoHistory.getRedoStack().push(redoState);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not serialize or hold references to EntityTransformSnapshot instances for long periods (e.g., across server restarts). The internal entity reference (`Ref<EntityStore>`) may become invalid, and the snapshot will be useless. It is designed for in-memory, session-scoped operations.
- **Cross-Thread Restoration:** As stated in the concurrency section, never call `restoreEntity` from an asynchronous task or any thread other than the main server thread. This will break ECS invariants.
- **State Modification:** Do not attempt to modify the internal state of a snapshot instance using reflection. The class's immutability is a core design guarantee. To capture a new state, create a new instance.

## Data Pipeline
EntityTransformSnapshot does not process a continuous stream of data. Instead, it acts as a control-flow component for discrete state capture and restoration events.

> Flow:
> User Action (e.g., Move Entity command) → **EntityTransformSnapshot (Creation & State Capture)** → Pushed to Undo Stack → User Input (Undo command) → Popped from Undo Stack → **EntityTransformSnapshot.restoreEntity()** → Entity Component State is Mutated in World


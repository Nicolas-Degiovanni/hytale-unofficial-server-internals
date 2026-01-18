---
description: Architectural reference for EntitySnapshot
---

# EntitySnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface EntitySnapshot<T extends SelectionSnapshot<?>> extends SelectionSnapshot<T> {
```

## Architecture & Concepts
The EntitySnapshot interface defines a strict contract for systems that capture and restore the state of entities within a defined game volume. It is a specialized component of the Builder Tools framework, designed to handle the data-oriented aspects of features like copy/paste, undo/redo, and area blueprinting.

This interface acts as a data transfer object (DTO) with behavior. It encapsulates the serialized data of one or more entities and provides the logic to re-instantiate them within a game World. Its primary responsibility is to bridge a point-in-time capture of entity state with the live, running server environment, ensuring that restoration operations are performed safely and correctly with respect to the game's main thread.

As a subtype of SelectionSnapshot, it integrates into a broader system for managing different types of world data (e.g., blocks, entities) in a uniform manner.

## Lifecycle & Ownership
As an interface, EntitySnapshot itself has no lifecycle. The following applies to concrete implementations of this contract.

-   **Creation:** Instances are typically created by a higher-level service, such as a SnapshotManager or a specific Builder Tool, in response to a player action (e.g., copying a selection). The instance captures the state of all entities within the target region at that moment.
-   **Scope:** The lifetime of an EntitySnapshot object is transient and tied to a specific user operation. It may be stored temporarily in a clipboard buffer, an undo/redo stack, or a blueprint library. It is not intended for long-term persistence across server restarts without further serialization.
-   **Destruction:** The object is eligible for garbage collection once it is no longer referenced by any system, such as after a paste operation is completed or an undo stack is cleared.

## Internal State & Concurrency
-   **State:** The interface is stateless. Concrete implementations are expected to be immutable data holders. Once an EntitySnapshot is created, the entity data it contains should not be modified. The restoration process reads from this state but does not alter it.
-   **Thread Safety:** The interface is designed to be thread-safe for restoration operations. The default `restore` method contains critical thread-dispatching logic. It detects if the caller is on the main World thread. If not, it schedules the actual restoration work (via `restoreEntity`) to be executed on the World thread using a CompletableFuture. This design centralizes concurrency control, freeing implementers of `restoreEntity` from writing boilerplate thread-safety code.

**WARNING:** Implementations of `restoreEntity` can and should assume they are executing on the main World thread. All external calls must go through the `restore` method to guarantee this safety.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| restoreEntity(Player, World, ComponentAccessor) | T | O(N) | **For internal use.** Restores the captured entities into the specified World. Assumes execution on the main World thread. N is the number of entities. |
| restore(Ref, Player, World, ComponentAccessor) | T | O(N) | **Primary entry point.** Safely restores entities by dispatching to the World thread if necessary. Blocks the calling thread until completion if called from an off-thread context. |

## Integration Patterns

### Standard Usage
The standard pattern involves a managing service obtaining an EntitySnapshot and invoking the `restore` method to apply it to the world. The system correctly handles the threading context.

```java
// A builder tool or manager holds a snapshot instance
EntitySnapshot<?> clipboard = player.getClipboard().getSnapshot();

if (clipboard != null) {
    // The restore method is the safe, public entry point
    // It handles dispatching to the world thread automatically
    clipboard.restore(null, player, player.getWorld(), null);
}
```

### Anti-Patterns (Do NOT do this)
-   **Directly Calling restoreEntity:** Never call `restoreEntity` directly from an arbitrary thread. The `restore` method provides essential thread-safety guarantees by scheduling the operation on the correct World thread. Bypassing it will lead to race conditions and world corruption.
-   **Assuming Immediate Execution:** When `restore` is called from an asynchronous task or a different thread, it schedules the work and blocks until it is complete. Do not design systems that depend on non-blocking behavior from this method in a multi-threaded context.

## Data Pipeline
The EntitySnapshot serves as a data payload in a command-oriented flow. It does not process a continuous stream of data but rather applies a discrete, self-contained dataset to the game world.

> Flow:
> Player Action (e.g., "Paste") -> Builder Tool Service -> **EntitySnapshot.restore()** -> World Thread Scheduler -> **EntitySnapshot.restoreEntity()** -> World.EntityStore Modification -> New Entities Spawned in World


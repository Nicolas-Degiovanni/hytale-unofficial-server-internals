---
description: Architectural reference for SelectionProvider
---

# SelectionProvider

**Package:** com.hypixel.hytale.server.core.prefab.selection
**Type:** Service Interface

## Definition
```java
// Signature
public interface SelectionProvider {
```

## Architecture & Concepts
The SelectionProvider interface defines a formal contract for systems that calculate a player's targeted selection within the game world. This is a fundamental capability for player interaction, forming the basis for actions such as breaking blocks, placing prefabs, or interacting with entities.

This is not a simple raycasting utility. It represents a high-level, asynchronous operation that decouples the *request* for a selection from the *computation* and *consumption* of the result. Implementations are expected to perform complex spatial queries against the world state, represented by the EntityStore, to determine the precise BlockSelection a Player is targeting.

The use of a ThrowableConsumer as a callback mechanism is a critical design choice. It allows the selection computation, which can be resource-intensive, to potentially be offloaded from the main game loop to prevent stalls or performance degradation. The caller provides the logic to execute upon completion, ensuring a non-blocking interaction pattern.

## Lifecycle & Ownership
As an interface, SelectionProvider itself has no lifecycle. The lifecycle is defined by its concrete implementations.

- **Creation:** Implementations of this interface are expected to be instantiated and managed by the server's central service registry or dependency injection framework at startup.
- **Scope:** A single, session-scoped instance of a SelectionProvider implementation typically exists for the duration of a server session. It is a shared, global service.
- **Destruction:** The instance is destroyed during server shutdown as part of the service registry cleanup.

## Internal State & Concurrency
- **State:** This interface is stateless by definition. However, implementations may be stateful, potentially caching spatial acceleration structures or other world data to optimize repeated queries. The contract guarantees that the result passed to the consumer is a *copy* (as implied by the method name computeSelectionCopy), protecting the caller from mutable internal state within the provider.
- **Thread Safety:** The interface contract is designed for concurrent access. Callers from any thread may invoke computeSelectionCopy.

    **WARNING:** There is no guarantee regarding the execution thread of the provided ThrowableConsumer. Implementations are free to execute the consumer on a worker thread or queue it for execution on the main server thread. Any logic within the consumer must be thread-safe or properly synchronized.

## API Surface
The public contract consists of a single method for requesting a selection calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSelectionCopy(var1, var2, var3, var4) | void | O(N) | Asynchronously computes the Player's target selection. The result is passed to the provided consumer. Complexity is dependent on view distance and world density. |

## Integration Patterns

### Standard Usage
Game logic systems, such as an interaction handler, should retrieve the active SelectionProvider from the server's service context and submit a request. The result is handled in a callback lambda.

```java
// A system retrieves the provider and requests a selection
SelectionProvider selectionProvider = serverContext.getService(SelectionProvider.class);
Player activePlayer = ...;
Ref<EntityStore> worldStore = ...;

selectionProvider.computeSelectionCopy(worldStore, activePlayer, (selection) -> {
    // This code executes when the selection is ready
    // It may be on a different thread
    if (selection.hasTarget()) {
        world.highlightBlock(selection.getTargetPosition());
    }
}, componentAccessor);
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations in Consumer:** Do not perform long-running or blocking operations inside the consumer lambda. This can stall the thread it executes on, which may be a shared worker thread, impacting overall server performance.
- **Assuming Synchronous Execution:** Do not write code that assumes the consumer is invoked before computeSelectionCopy returns. The operation is explicitly asynchronous.
- **Direct Implementation:** Game feature developers should consume this interface, not implement it. Implementing SelectionProvider is reserved for core engine systems that define new methods of world intersection and querying.

## Data Pipeline
The interface facilitates a one-way data flow from a request to an asynchronous result.

> Flow:
> Player Input (Camera Vector) -> Game System -> **SelectionProvider.computeSelectionCopy** -> [Internal Raycast/Voxel Traversal] -> BlockSelection Object -> ThrowableConsumer Callback -> Game System Logic (e.g., Block Highlighting, Damage Application)


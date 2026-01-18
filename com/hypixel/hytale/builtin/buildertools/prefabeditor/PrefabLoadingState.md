---
description: Architectural reference for PrefabLoadingState
---

# PrefabLoadingState

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Transient State Object

## Definition
```java
// Signature
public class PrefabLoadingState {
```

## Architecture & Concepts
The PrefabLoadingState class is a state machine and data container designed to track the progress of a long-running, multi-stage prefab loading operation. It is a critical component of the server-side builder tools, specifically for the prefab editor feature.

Its primary architectural role is to **decouple the asynchronous prefab processing logic from the user-facing feedback systems**. A worker process, responsible for file I/O and world modification, mutates an instance of this class to report its progress. Concurrently, a separate system, such as a command handler or UI controller, reads from the same instance to display status messages, progress bars, and errors to the player.

This class does not perform any work itself; it is a passive data structure that models the state of a complex workflow, including initialization, world creation, asset loading, world pasting, finalization, and potential cancellation or error states.

## Lifecycle & Ownership
-   **Creation:** An instance of PrefabLoadingState is created on-demand when a prefab editing session is initiated. This is typically triggered by a server command like `/editprefab`. The command handler or a dedicated `PrefabEditorService` is responsible for its instantiation.
-   **Scope:** The object's lifetime is strictly bound to a single, continuous prefab loading operation. It persists from the moment the operation begins until it concludes in a terminal state (COMPLETE, ERROR, SHUTDOWN_COMPLETE).
-   **Destruction:** The object is not explicitly destroyed. It becomes eligible for garbage collection once all references to it are released. This occurs after the loading operation has finished and any associated UI elements have been dismissed. A new operation requires a new instance.

## Internal State & Concurrency
-   **State:** This object is highly **mutable**. Its core purpose is to have its internal fields (such as `currentPhase`, `loadedPrefabs`, and `errors`) updated sequentially as the loading process advances. It caches progress counters and a history of errors.

-   **Thread Safety:** **CRITICAL:** This class is **not thread-safe**. It contains no internal synchronization mechanisms like locks or volatile fields. All methods directly manipulate shared state. The responsibility for ensuring safe concurrent access lies entirely with the calling systems. Typically, a single worker thread writes to the instance, while a single main server thread reads from it. Any deviation from this pattern requires external synchronization (e.g., `synchronized` blocks) around all read and write operations to prevent race conditions, memory visibility issues, and inconsistent state reads.

## API Surface
The public API is designed for a clear separation between state mutation (by the worker) and state consumption (by the UI layer).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setPhase(Phase phase) | void | O(1) | Advances the state machine to a new phase. Central to the workflow. |
| onPrefabLoaded(Path path) | void | O(1) | Increments the loaded prefab counter. Called repeatedly by the worker. |
| onPrefabPasted(Path path) | void | O(1) | Increments the pasted prefab counter. Called repeatedly by the worker. |
| addError(String key, String details) | void | O(1) | Adds a loading error and immediately transitions the state machine to the ERROR phase. |
| getProgressPercentage() | float | O(1) | Calculates a normalized progress value (0.0 to 1.0) based on the current phase and counters. |
| getStatusMessage() | Message | O(1) | Constructs a translatable message suitable for display to the player, reflecting the current status or error. |
| isShuttingDown() | boolean | O(1) | Returns true if the operation is in a cancellation or shutdown phase. |
| hasErrors() | boolean | O(1) | A quick check to see if the operation has failed. |

## Integration Patterns

### Standard Usage
The intended pattern involves a controller creating the state object, passing it to a worker, and polling it for UI updates.

```java
// 1. A command handler or service initiates the process
PrefabLoadingState state = new PrefabLoadingState();
player.getUI().showLoadingScreen(state); // UI subscribes to state changes

// 2. A separate worker thread is given the state object
PrefabLoadTask task = new PrefabLoadTask(world, prefabs, state);
server.getExecutor().submit(task);

// Inside PrefabLoadTask.run()...
state.setPhase(Phase.LOADING_PREFABS);
for (Path prefab : prefabs) {
    // ... load logic ...
    state.onPrefabLoaded(prefab);
}
// ... more state updates ...
state.markComplete();
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not reuse a PrefabLoadingState instance for a new loading operation. Its internal counters and timestamps are bound to its initial creation. Always create a `new PrefabLoadingState()` for each new session.
-   **Unsynchronized Concurrent Access:** Do not allow multiple threads to write to the state object simultaneously. More importantly, do not read from the object on one thread while another thread is writing without a proper memory barrier or lock. This will lead to inconsistent UI (e.g., progress bar jumping backwards) and potential crashes.
-   **External State Modification:** Do not modify the state from outside the designated worker process after the operation has started. The state should be a one-way flow of information from the worker to the rest of the system.

## Data Pipeline
PrefabLoadingState acts as a shared memory buffer to communicate progress from a background task to a user-facing system.

> Flow:
> Server Command -> **PrefabLoadingState (Instantiation)** -> Async Worker Task (Mutates State) -> Server Tick/UI Thread (Polls State) -> Player UI Update (Progress Bar, Status Message)


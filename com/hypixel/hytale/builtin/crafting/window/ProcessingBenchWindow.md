---
description: Architectural reference for ProcessingBenchWindow
---

# ProcessingBenchWindow

**Package:** com.hypixel.hytale.builtin.crafting.window
**Type:** Transient

## Definition
```java
// Signature
public class ProcessingBenchWindow extends BenchWindow implements ItemContainerWindow {
```

## Architecture & Concepts

The ProcessingBenchWindow is a server-side state object that represents a player's active user interface for a processing-style crafting station, such as a furnace, smelter, or alloy forge. It acts as the critical bridge between the persistent world state, represented by a ProcessingBenchState, and the transient, per-player UI session.

This class is not a component in the Entity-Component-System (ECS) sense. Instead, it is a dedicated view model for the network layer. Its primary responsibility is to serialize the state of a processing bench (e.g., fuel level, crafting progress, active status) into a JSON format suitable for consumption by the client. It also serves as the server's entry point for handling UI-specific actions initiated by the player, such as activating the bench or triggering a tier upgrade.

The core of its state is maintained in a JsonObject named windowData. Any mutation to the window's state, such as a call to setProgress or setActive, updates this JsonObject and flags the window as dirty by calling invalidate. The server's window management system is then responsible for synchronizing this updated JSON data with the connected client.

## Lifecycle & Ownership

-   **Creation:** A ProcessingBenchWindow is instantiated by the server's window management system when a player successfully opens a block entity that has an associated ProcessingBenchState. The initial state of the window is hydrated directly from the corresponding ProcessingBenchState.

-   **Scope:** The object's lifetime is strictly tied to the player's UI session with the specific bench. It persists only as long as the player has the processing bench window open on their client.

-   **Destruction:** The instance is marked for garbage collection when the player closes the UI window. The onClose0 method is invoked as part of this process, ensuring that any registered event listeners, such as the inventory change listener, are properly unregistered to prevent memory leaks.

## Internal State & Concurrency

-   **State:** This object is highly mutable. It maintains the real-time state of the UI, including fuel time, progress, active status, and bitmasks for currently processing slots. All of this state is aggregated into the internal windowData JsonObject, which serves as the single source of truth for the client.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the primary world update thread. All interactions, whether from network packet handling or from game logic updating the bench's state, are expected to be synchronized by the server's main game loop. Unsynchronized access from other threads will lead to race conditions, corrupted state, and client desynchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handleAction(ref, store, action) | void | O(1) | Entry point for processing network packets from the client. Dispatches actions like SetActiveAction to modify the underlying bench state. |
| setActive(boolean active) | void | O(1) | Sets the active processing status. Triggers a network update via invalidate. |
| setFuelTime(float fuelTime) | void | O(1) | Updates the remaining fuel time. Throws IllegalArgumentException for infinite or NaN values. |
| setProgress(float progress) | void | O(1) | Updates the current crafting progress percentage. Throws IllegalArgumentException for infinite or NaN values. |
| setProcessingSlots(Set slots) | void | O(N) | Updates the set of input slots currently being processed. Recalculates and stores a bitmask for network efficiency. |
| updateBenchTierLevel(int level) | void | O(N) | Reconfigures the window's layout and item container when the underlying bench is upgraded. |

## Integration Patterns

### Standard Usage

This class is an internal implementation detail of the crafting and windowing systems. Direct interaction is typically reserved for the underlying state object or the network packet handler. Game logic should modify the ProcessingBenchState, which in turn propagates changes to this window.

```java
// Conceptual example of how the system updates the window
// This code would exist within the ProcessingBenchState update logic.

// Assume 'player' has this window open
ProcessingBenchWindow window = player.getOpenWindow(ProcessingBenchWindow.class);

if (window != null && window.isForBench(this)) {
    // The state machine for the bench calculates new progress
    float newProgress = calculateCurrentProgress();
    window.setProgress(newProgress);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ProcessingBenchWindow()`. The window management system is solely responsible for its creation and lifecycle. Manual instantiation will result in a disconnected object that is not registered to any player and will not function.
-   **State Caching:** Do not cache a reference to this object. Its lifetime is ephemeral. Always retrieve the current window instance from the Player object when needed.
-   **Asynchronous Modification:** Do not modify the window's state from a separate thread. All calls to setters like setProgress or setActive must originate from the main server thread to prevent data corruption.

## Data Pipeline

The ProcessingBenchWindow facilitates a bidirectional data flow between the server's game state and the player's client.

**Server State to Client UI:**

> Flow:
> ProcessingBenchState Tick -> Logic Update -> **ProcessingBenchWindow.setProgress()** -> windowData JsonObject Modified -> invalidate() -> Window System -> Network Packet -> Client UI Render

**Client Action to Server State:**

> Flow:
> Player Clicks UI Button -> Client Sends SetActiveAction Packet -> Server Network Layer -> **ProcessingBenchWindow.handleAction()** -> ProcessingBenchState.setActive() -> World State Change


---
description: Architectural reference for BenchWindow
---

# BenchWindow

**Package:** com.hypixel.hytale.builtin.crafting.window
**Type:** Transient State Object

## Definition
```java
// Signature
public abstract class BenchWindow extends BlockWindow implements MaterialContainerWindow {
```

## Architecture & Concepts
The BenchWindow is an abstract, server-side state model that represents a player's active interaction with a crafting bench block in the world. It is not a UI component itself, but rather the authoritative data source that the client-side user interface binds to.

This class acts as a bridge between the core crafting logic, managed by the CraftingManager, and the player's client. Its primary responsibility is to maintain the state of the crafting session—including progress, tier levels, and available resources—and to efficiently synchronize this state with the client.

As a subclass of BlockWindow, its lifecycle is intrinsically tied to a specific block's location in the world. As an implementation of MaterialContainerWindow, it participates in the system for managing and displaying required crafting materials.

## Lifecycle & Ownership
The lifecycle of a BenchWindow instance is ephemeral and strictly bound to a player's UI session with a specific bench.

-   **Creation:** An instance of a concrete BenchWindow subclass is created by the server's window management system when a player successfully interacts with a compatible bench block. The corresponding BenchState, which represents the persistent data of the block entity, is provided during construction.
-   **Scope:** The object exists only for the duration that the player has the specific bench UI open. All state within this object, such as crafting progress, is considered transient session data.
-   **Destruction:** The object is marked for garbage collection when the player closes the window. The `onClose0` method is invoked as a final cleanup hook, ensuring the CraftingManager is notified to terminate any associated crafting jobs and release resources.

## Internal State & Concurrency
-   **State:** The BenchWindow is highly mutable. Its core state is stored in the `windowData` JsonObject, which is frequently updated with crafting progress, tier levels, and other metadata. It also maintains internal state for update throttling, such as `lastUpdatePercent` and `lastUpdateTimeMs`, to prevent flooding the client with network packets. The `extraResourcesSection` serves as a cache for material data.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be accessed exclusively by the primary server thread responsible for the associated player or world tick. All interactions, state mutations, and API calls must originate from this synchronized context. Unmanaged multi-threaded access will lead to race conditions, particularly with the update-throttling logic and the `windowData` object.

## API Surface
The public API provides hooks for other systems to drive the state of the crafting UI.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| updateCraftingJob(float percent) | void | O(1) | Updates the crafting progress bar. Throttles network updates. |
| updateBenchUpgradeJob(float percent) | void | O(1) | Updates the tier upgrade progress bar. Throttles network updates. |
| updateBenchTierLevel(int newValue) | void | O(1) | Sets the new tier level of the bench and forces a UI rebuild. |
| invalidateExtraResources() | void | O(1) | Marks the cached material data as stale, forcing a refresh on next access. |
| onOpen0() | boolean | O(1) | Lifecycle hook. Registers the active bench with the player's CraftingManager. |
| onClose0() | void | O(1) | Lifecycle hook. Clears the active bench from the CraftingManager. |

## Integration Patterns

### Standard Usage
The BenchWindow is not meant to be controlled directly by gameplay code. It is a passive object managed by higher-level systems like the CraftingManager. The manager performs the crafting logic and pushes state changes to the window.

```java
// Example from a hypothetical CraftingSystem
// This code would not be written by a typical modder.

// Retrieve the player's currently open window
BenchWindow currentBench = player.getWindow(BenchWindow.class);

if (currentBench != null) {
    // Update the UI with the job's progress
    float progress = craftingJob.getPercentComplete();
    currentBench.updateCraftingJob(progress);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BenchWindowSubclass()`. The server's windowing framework is solely responsible for creating and managing window lifecycles. Manual creation will result in a disconnected object that is not linked to any player.
-   **State Caching:** Do not cache a reference to a BenchWindow instance across ticks. Its lifetime is volatile; it can be destroyed at any moment if the player closes the UI. Always retrieve the current window from the player entity each tick if needed.
-   **Asynchronous Modification:** Do not modify a BenchWindow from an asynchronous task or a different thread. All mutations must occur on the main server thread to prevent data corruption and race conditions.

## Data Pipeline
The BenchWindow is a critical node in the data flow from server-side logic to the client-side UI renderer.

> Flow:
> CraftingManager (calculates progress) -> **BenchWindow.updateCraftingJob()** -> Internal state update (`windowData`) -> **BenchWindow.invalidate()** -> Server Window System (detects invalidation) -> Network Packet Serialization -> Client Network Handler -> Client UI State Update -> UI Render


---
description: Architectural reference for MemoriesWindow
---

# MemoriesWindow

**Package:** com.hypixel.hytale.builtin.adventure.memories.window
**Type:** Transient State Object

## Definition
```java
// Signature
public class MemoriesWindow extends Window {
```

## Architecture & Concepts
The MemoriesWindow class is a server-side data model that represents the "Memories" user interface for a specific player. It is not a UI component itself, but rather a data provider that serializes a player's collected memories into a JSON payload destined for the client.

This class acts as a critical bridge between the persistent entity component system (specifically the PlayerMemories component) and the client's rendering layer. Its primary function is to be invoked by the server's windowing framework, transform raw game data into a client-consumable format, and then be discarded. It is a single-purpose, short-lived object that facilitates the display of dynamic, player-specific data.

## Lifecycle & Ownership
- **Creation:** An instance of MemoriesWindow is created on-demand by the server's internal WindowManager when a player entity requests to open a window of type `WindowType.Memories`. This is an indirect, framework-driven instantiation; developers do not create this object manually.
- **Scope:** The object's lifecycle is strictly bound to the duration that the corresponding UI is open for a single player. It persists only for that specific viewing session.
- **Destruction:** The object is marked for garbage collection immediately after the player closes the Memories window or disconnects from the server. The `onClose0` method is invoked as a final cleanup hook just prior to its logical destruction.

## Internal State & Concurrency
- **State:** The core state is a private, final JsonObject field named `windowData`. While the reference to this object is final, the object itself is **highly mutable**. Its content is constructed dynamically and entirely within the `onOpen0` lifecycle method. It serves as a temporary buffer for the JSON payload before it is sent to the client.
- **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. All interactions, from creation to method invocation, are expected to occur exclusively on the owning player's dedicated entity-processing thread. Any access from other threads will result in severe concurrency violations and unpredictable behavior.

## API Surface
The public contract is defined by its parent `Window` class and is centered around lifecycle hooks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onOpen0() | boolean | O(N * C) | Populates the internal JSON state from the player's `PlayerMemories` component. Complexity is N (player memories) times C (total memory categories). Triggers a data invalidation to sync with the client. |
| getData() | JsonObject | O(1) | Returns the fully constructed JSON payload for the client. This should only be called by the framework after `onOpen0` has completed. |
| onClose0() | void | O(1) | A no-op lifecycle hook for cleanup. This window requires no special teardown logic. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is not a standard development pattern. Instead, server-side logic triggers the windowing system to open the UI for a player, which in turn instantiates and uses this class under the hood.

```java
// CONCEPTUAL: This logic resides within the server framework, not user code.
// A player entity interacts with an object or command, which triggers this.

PlayerEntity player = getPlayerFromContext();
player.getWindowManager().openWindow(WindowType.Memories);

// The framework then finds, instantiates, and runs the MemoriesWindow lifecycle.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MemoriesWindow()`. The object is inert and useless without being managed by a player's `WindowManager`, which provides the necessary context like the player reference.
- **State Mutation After Opening:** Modifying the state of this object after the `onOpen0` method has returned is undefined behavior. The data is serialized and sent to the client during the `invalidate` call within `onOpen0`; subsequent changes will be ignored.
- **Reusing Instances:** A MemoriesWindow instance is single-use. It is created for one "open" event and destroyed on "close". Do not attempt to cache or reuse these objects.

## Data Pipeline
The primary role of this class is to act as a transformation stage in a data pipeline that flows from the server's persistent storage to the player's screen.

> Flow:
> PlayerMemories Component (BSON Data) -> `onOpen0()` Transformation -> **MemoriesWindow** (In-Memory JSON) -> `invalidate()` -> Server Network Layer -> Client UI Renderer


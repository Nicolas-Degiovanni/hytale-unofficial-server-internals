---
description: Architectural reference for MemoriesPage
---

# MemoriesPage

**Package:** com.hypixel.hytale.builtin.adventure.memories.page
**Type:** Transient

## Definition
```java
// Signature
public class MemoriesPage extends InteractiveCustomUIPage<MemoriesPage.PageEventData> {
```

## Architecture & Concepts

The MemoriesPage class is a server-side controller that manages the player-facing user interface for the Memories system. It extends InteractiveCustomUIPage, integrating it deeply into the server's UI framework. This framework operates by having the server define UI structures and logic, which are then sent to the client for rendering. MemoriesPage does not render pixels; it builds a declarative representation of the UI that the client interprets.

Architecturally, this class acts as the Controller in a server-side MVC pattern:
*   **Model:** The underlying data is sourced from two primary locations: the global singleton `MemoriesPlugin` (containing all possible memories and globally recorded ones) and the player-specific `PlayerMemories` component (containing newly discovered memories for that player).
*   **View:** The view is defined dynamically within the `build` method using a `UICommandBuilder`. This builder constructs a series of commands that describe the UI layout, text, and assets, which are then serialized and sent to the client.
*   **Controller:** The class itself serves as the controller. It manages the UI's internal state (such as the currently selected category or memory) and responds to client-side interactions via the `handleDataEvent` method.

The class is responsible for rendering two distinct views: a main category overview and a detailed view for memories within a selected category. It transitions between these views by modifying its internal state and triggering a UI rebuild.

## Lifecycle & Ownership

-   **Creation:** An instance of MemoriesPage is created when a player interacts with a specific in-world entity or block configured to open the Memories UI. The constructor requires a `PlayerRef` and the `BlockPosition` of the interaction point, which is later used for positioning particle effects. It is not managed by a dependency injection framework and is instantiated directly by gameplay code.
-   **Scope:** The object's lifetime is strictly bound to the player's interaction with the UI screen. It persists only as long as the Memories page is open on the client. The `CustomPageLifetime.CanDismissOrCloseThroughInteraction` configuration confirms that its lifecycle is managed by the player's direct UI actions.
-   **Destruction:** The instance is marked for garbage collection when the player closes the UI (e.g., by pressing the escape key) or when the `close()` method is invoked programmatically. A primary trigger for programmatic closure is the successful recording of memories via the "Record" button.

## Internal State & Concurrency

-   **State:** MemoriesPage is highly stateful and mutable. Its internal state dictates which UI panel is rendered. Key state fields include:
    *   `currentCategory`: A nullable String. If null, the top-level category selection panel is built. If non-null, the memory list for that category is built.
    *   `selectedMemory`: A nullable Memory object. Stores the memory currently being inspected by the player in the detailed view.
    *   `recordMemoriesParticlesPosition`: A Vector3d set during construction. This state is immutable after creation and is used as a world-space anchor for visual effects.

-   **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. All methods are designed to be called exclusively from the server thread responsible for the associated player's game logic. The server's entity and UI processing model ensures single-threaded access, preventing race conditions. Any attempt to access or modify an instance of this class from an asynchronous task or a different thread will result in undefined behavior and severe state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | **Framework Callback.** Populates the UI command buffer based on internal state. Complexity is proportional to the number of memory categories or memories in the current view. |
| handleDataEvent(ref, store, data) | void | O(N) | **Framework Callback.** Entry point for handling all user interactions from the client. Mutates internal state and triggers UI updates. Complexity depends on the action, potentially iterating over memories to find a match. |

## Integration Patterns

### Standard Usage

This class is not intended to be retrieved from a service registry. It is instantiated and managed by gameplay logic that handles player interactions. The standard lifecycle is initiated by an interaction handler.

```java
// In a block interaction handler or similar gameplay code:
PlayerRef playerRef = ...;
BlockPosition blockPosition = ...;

// 1. Create a new instance for the specific interaction.
MemoriesPage page = new MemoriesPage(playerRef, blockPosition);

// 2. Open the page for the player. The framework takes over from here.
player.getPages().open(page);
```

### Anti-Patterns (Do NOT do this)

-   **Direct State Manipulation:** Never modify fields like `currentCategory` or `selectedMemory` from outside the class. All state transitions must be driven by events processed through `handleDataEvent` to ensure the UI remains synchronized with the state.
-   **Instance Caching:** Do not cache and reuse `MemoriesPage` instances across multiple UI openings. A new instance must be created each time the player opens the interface to guarantee a clean initial state.
-   **Manual Invocation of `build`:** The `build` method is a callback controlled by the UI framework. Calling it directly will not affect the client's UI and serves no purpose. To force a UI update, use the `rebuild()` or `sendUpdate()` methods.

## Data Pipeline

The flow of data is bidirectional, involving an initial render pass and subsequent event-driven updates.

> **UI Render Flow:**
> `MemoriesPage.build()` → Reads `MemoriesPlugin` & `PlayerMemories` → `UICommandBuilder` → Server UI System → Network Packet → Client UI Renderer

> **Client Event Flow:**
> Client Button Click → Network Packet (containing `PageEventData`) → Server UI System → `MemoriesPage.handleDataEvent()` → State Mutation → Triggers `rebuild()` or `sendUpdate()` → UI Render Flow
---


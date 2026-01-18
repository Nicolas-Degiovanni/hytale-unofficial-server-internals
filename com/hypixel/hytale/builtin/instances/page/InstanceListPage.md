---
description: Architectural reference for InstanceListPage
---

# InstanceListPage

**Package:** com.hypixel.hytale.builtin.instances.page
**Type:** Stateful Component

## Definition
```java
// Signature
public class InstanceListPage extends InteractiveCustomUIPage<InstanceListPage.PageData> {
```

## Architecture & Concepts
The InstanceListPage class is a server-side controller that manages the "Instance Selection" user interface. It follows a Model-View-Controller (MVC) pattern where this class acts as the Controller.

-   **Model:** The underlying data is the list of available instance assets, provided by the global InstancesPlugin singleton.
-   **View:** The client-side UI, defined by the `Pages/InstanceListPage.ui` and `Pages/BasicTextButton.ui` asset files. This class does not contain rendering logic; it only sends commands to manipulate the client-side view.
-   **Controller:** This class bridges player input with server logic. It builds the initial list of instances, handles button clicks for selection, and executes high-level actions like spawning or loading an instance.

It is fundamentally an event-driven component, reacting to `CustomUIEventBindingType` events sent from the client and translating them into world-altering operations. The `PageData` inner class serves as the Data Transfer Object (DTO) for these events, deserialized from the network protocol.

## Lifecycle & Ownership
-   **Creation:** An InstanceListPage is instantiated on-demand when a player needs to be shown the instance selection screen. It is created with a reference to a specific player (PlayerRef), tying its lifecycle directly to that player's UI session. The creator is typically a higher-level game logic system, such as a command handler or interactive block.

-   **Scope:** The object's lifetime is ephemeral, existing only as long as the UI page is visible to the player. It is not intended to persist across sessions or even long-term within a single session.

-   **Destruction:** The instance is marked for garbage collection when the UI is closed. This happens in one of three ways:
    1.  The player manually dismisses the UI (as permitted by `CustomPageLifetime.CanDismiss`).
    2.  A terminal action like **Load** or **Spawn** is successfully triggered, which programmatically calls the `close()` method.
    3.  The player disconnects, and the server's UI management system cleans up all associated pages.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains the list of all displayed instances (`instances`) and the currently selected one (`selectedInstance`). This state is modified exclusively in response to player interaction events.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the primary world thread. All world-mutating operations, such as `load` and `spawn`, explicitly schedule their logic onto the world's execution context using `world.execute()`. Asynchronous operations, like loading an instance asset, use `CompletableFuture` to avoid blocking the main thread, with callbacks marshaled back to the world thread for safe state mutation.

    **WARNING:** Calling methods on this class from an external or worker thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is defined by its inheritance from `InteractiveCustomUIPage`. Direct invocation of these methods is managed by the server's UI framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Called once by the UI framework to populate the initial state of the page. Iterates through all available instances (N) to build the button list. |
| handleDataEvent(ref, store, data) | void | O(1) | The primary event handler. Called by the UI framework when the client sends an event. Dispatches actions based on the incoming PageData. |

## Integration Patterns

### Standard Usage
This class is not meant to be used like a typical service. It is instantiated and passed to the server's UI manager, which then takes ownership of its lifecycle.

```java
// Example: A command handler opening the instance page for a player.
// Note: UIManager is a hypothetical service for this example.

PlayerRef playerRef = ...; // Get the player reference
Player player = ...; // Get the player entity

// The UI framework takes ownership of the new page instance.
player.getUIManager().openPage(new InstanceListPage(playerRef));
```

### Anti-Patterns (Do NOT do this)
-   **State Caching:** Do not retain a reference to an InstanceListPage instance after passing it to the UI framework. Its lifecycle is short and managed internally. Attempting to interact with a closed page will have no effect or throw exceptions.
-   **Direct Event Handling:** Do not call `handleDataEvent` manually. This method is the entry point for deserialized network packets and should only be invoked by the UI event routing system.
-   **Manual State Updates:** Do not modify `selectedInstance` or other internal fields directly. All state changes must flow through the `handleDataEvent` method to ensure the client UI is correctly updated via `sendUpdate`.

## Data Pipeline
The class operates within a bidirectional data flow between the client and server.

> **Client to Server (Player Action):**
> Player Click -> Client UI sends `CustomUIEvent` Packet -> Server Network Layer -> UI Framework decodes `PageData` -> **InstanceListPage.handleDataEvent()** -> Server Logic (e.g., `spawn`, `load`)

> **Server to Client (UI Update):**
> **InstanceListPage.updateSelection()** -> `UICommandBuilder` populates commands -> `sendUpdate()` -> UI Framework serializes commands -> Server Network Layer sends `CustomUICommand` Packet -> Client UI receives commands -> Client-side elements are updated (e.g., button style changes)


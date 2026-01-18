---
description: Architectural reference for CommandListPage
---

# CommandListPage

**Package:** com.hypixel.hytale.server.core.command.system.pages
**Type:** Stateful Component

## Definition
```java
// Signature
public class CommandListPage extends InteractiveCustomUIPage<CommandListPage.CommandListPageEventData> {
```

## Architecture & Concepts

The CommandListPage class is a server-side controller for the interactive command helper user interface. It embodies a server-authoritative UI pattern, where the server holds the complete state and logic, sending declarative updates to the client for rendering. This class does not render anything directly; instead, it generates instructions for the client via UICommandBuilder and UIEventBuilder.

Architecturally, it serves as a bridge between the CommandManager (the registry of all server commands) and a specific player's client UI. For each player that opens the command list, a dedicated instance of CommandListPage is created. This instance maintains the UI state for that player, such as the current search query, the selected command, and navigation history (breadcrumbs).

The core responsibility of this class is to:
1.  Query the CommandManager for commands available to the current player, respecting permissions.
2.  Build the initial UI layout and data based on the game's UI templates (e.g., `Pages/CommandListPage.ui`).
3.  Receive and process user interaction events (e.g., clicks, text input) sent from the client.
4.  Update its internal state in response to these events.
5.  Generate and transmit differential UI updates back to the client, ensuring the client's view is synchronized with the server's state.

A notable and high-risk implementation detail is its heavy reliance on Java Reflection to access private fields within AbstractCommand (e.g., `variantCommands`, `requiredArguments`). This creates a fragile coupling; changes to the AbstractCommand class are likely to break CommandListPage without compile-time errors.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand when a player needs to view the command UI. This is typically triggered by another system, such as a command execution block, which then assigns the new page to the player via their `PageManager`.
    ```java
    // Example instantiation
    player.getPageManager().setPage(ref, store, new CommandListPage(player.getRef()));
    ```
-   **Scope:** The object's lifetime is tied directly to the player's interaction with this specific UI. It persists as long as the command list page is the active page for the player.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the player's `PageManager` sets a new page or closes the current one. The `handleSendToChat` method is a self-terminating pathway, explicitly closing the page after its action is complete.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains the complete view state of the UI for a single player, including `searchQuery`, `selectedCommand`, `subcommandBreadcrumb`, and a cached list of `visibleCommands`. All fields are private and managed internally in response to events.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to be managed exclusively by the server's entity processing thread responsible for the associated player. The Hytale server's Entity Component System (ECS) guarantees that all interactions with a player's components and associated objects like this page are serialized onto a single thread, thus preventing race conditions without requiring explicit locks within this class.

## API Surface

The public contract is defined by its role as an InteractiveCustomUIPage.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CommandListPage(playerRef) | constructor | O(1) | Creates a command page for a player. |
| CommandListPage(playerRef, initialCommand) | constructor | O(1) | Creates a command page, attempting to select a specific command on open. |
| build(ref, commandBuilder, eventBuilder, store) | void | O(C log C) | Called once by the PageManager to construct the initial UI. Complexity is dominated by fetching, filtering, and sorting all available commands (C). |
| handleDataEvent(ref, store, data) | void | O(C) | The primary entry point for all client interactions. Dispatches to internal handlers based on event data. Complexity can vary; search-related events re-run the command filtering logic. |

## Integration Patterns

### Standard Usage

The CommandListPage is intended to be instantiated and managed by the player's PageManager. A system or command handler creates an instance and sets it as the player's active page.

```java
// A player executes a command that opens the UI
Player playerComponent = store.getComponent(ref, Player.getComponentType());
if (playerComponent != null) {
    // The PageManager takes ownership of the new CommandListPage instance
    playerComponent.getPageManager().setPage(ref, store, new CommandListPage(playerRef));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct State Mutation:** Do not modify internal state fields like `selectedCommand` or `searchQuery` from outside the class. All state changes must be driven through the `handleDataEvent` method to ensure the UI remains synchronized.
-   **Instance Re-use:** A CommandListPage instance is bound to a single player via its `playerRef`. Never re-use an instance for a different player.
-   **Manual UI Building:** Do not call the `build` method directly. The framework's PageManager is responsible for invoking the build lifecycle at the appropriate time. Manually calling it will result in a detached or inconsistent UI state.

## Data Pipeline

The component operates within a bidirectional data flow between the client and server.

> **Client -> Server (Input)**:
> Client UI Interaction (e.g., Button Click) -> `CustomUIEvent` Network Packet -> Server's Player Packet Handler -> `InteractiveCustomUIPage.handleDataEvent` -> **CommandListPage.handleDataEvent(data)**

> **Server -> Client (Output)**:
> **CommandListPage** (updates state and populates builders) -> `UICommandBuilder` & `UIEventBuilder` -> `sendUpdate()` -> `CustomUIPageUpdate` Network Packet -> Client Network Handler -> Client UI Engine -> Visual elements are updated on screen.


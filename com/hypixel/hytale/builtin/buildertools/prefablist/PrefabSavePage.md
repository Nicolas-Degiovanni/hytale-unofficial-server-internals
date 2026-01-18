---
description: Architectural reference for PrefabSavePage
---

# PrefabSavePage

**Package:** com.hypixel.hytale.builtin.buildertools.prefablist
**Type:** Transient

## Definition
```java
// Signature
public class PrefabSavePage extends InteractiveCustomUIPage<PrefabSavePage.PageData> {
```

## Architecture & Concepts
The PrefabSavePage class is a server-side controller that manages the "Save Prefab" user interface dialog for a specific player. It follows a remote controller pattern where the server defines the UI structure and event bindings, sends them to the client for rendering, and processes data returned from client-side interactions.

This class acts as a critical bridge between player input and the core builder tools system. Its primary responsibilities are:
1.  **UI Definition:** Declaratively building the UI's initial state and wiring client-side events (like button clicks) to server-side data payloads using the `build` method.
2.  **Data Contract:** Defining the structure of data sent from the client via the nested PageData class, which serves as a Data Transfer Object (DTO).
3.  **Event Handling:** Receiving and processing the PageData payload in the `handleDataEvent` method.
4.  **Input Validation:** Performing initial validation on user input, such as ensuring a prefab name is provided.
5.  **Delegation:** Decoupling UI logic from world-modification logic by delegating the actual save operation to the BuilderToolsPlugin's asynchronous task queue.

This component is fundamental to the builder tools feature, providing the necessary user interaction layer for persisting in-game creations.

### Lifecycle & Ownership
-   **Creation:** An instance of PrefabSavePage is created and activated by a server-side system when a player needs to save a prefab. This is achieved by calling `player.getPageManager().setPage(ref, store, new PrefabSavePage(playerRef))`.
-   **Scope:** The object's lifetime is strictly tied to the visibility of the "Save Prefab" UI for a single player. It persists only as long as the dialog is open on the client's screen.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection as soon as the UI is closed. This is triggered explicitly within the `handleDataEvent` method by calling `playerComponent.getPageManager().setPage(ref, store, Page.None)` after a successful save or cancellation.

## Internal State & Concurrency
-   **State:** This class is designed to be effectively stateless between requests. Its only member field, `playerRef`, is an immutable reference set at construction. All dynamic state, such as the contents of the name input field or the status of checkboxes, is maintained exclusively on the client. This state is only made available to the server within the scope of the `handleDataEvent` method via the PageData parameter.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. All interactions are managed by the player's PageManager, which guarantees that its methods are invoked sequentially on the appropriate server thread for that player. Asynchronous or heavy operations are correctly offloaded to the BuilderToolsPlugin queue to avoid blocking the player's update loop.

## API Surface
The public contract is defined by its role as an InteractiveCustomUIPage. Direct invocation of these methods is managed by the engine's UI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(1) | Called by the engine to construct the UI commands and event bindings that are sent to the client. |
| handleDataEvent(ref, store, data) | void | O(1) | Callback executed when the client sends a UI event. Contains the core logic for validation and delegating the save action. |

## Integration Patterns

### Standard Usage
This page should always be instantiated and immediately passed to the player's PageManager to be displayed.

```java
// Example of a command handler opening the Prefab Save UI for a player.
// Assume 'player' and 'playerRef' are available in the current context.

// The PageManager takes ownership and manages the lifecycle of the new page.
player.getPageManager().setPage(ref, store, new PrefabSavePage(playerRef));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation without Management:** Do not create an instance of PrefabSavePage without passing it to a PageManager. The object will be orphaned and will have no effect.
-   **Performing Blocking Operations:** Do not execute long-running or blocking tasks (file I/O, heavy computations, direct world edits) inside `handleDataEvent`. This will stall the player's processing thread. The correct pattern, as implemented, is to delegate these tasks to a dedicated, asynchronous system like the BuilderToolsPlugin queue.
-   **Adding Mutable State:** Do not add mutable member fields to this class to track UI state. The client is the single source of truth for the UI's state. Rely solely on the data provided in the `handleDataEvent` payload.

## Data Pipeline
The flow of data and control for this component is unidirectional and follows a clear request-response cycle between the client and server.

> Flow:
> Server Command -> **PrefabSavePage Instantiation** -> `build()` method generates UI commands -> Network Packet to Client -> Client Renders UI -> Player Interaction -> Client sends `CustomUIEvent` with PageData payload -> Network Packet to Server -> Engine routes to `handleDataEvent()` -> **PrefabSavePage** validates and delegates to BuilderToolsPlugin -> **PrefabSavePage** sends command to close UI


---
description: Architectural reference for CustomUIPage
---

# CustomUIPage

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages
**Type:** Transient State Object

## Definition
```java
// Signature
public abstract class CustomUIPage {
```

## Architecture & Concepts

The CustomUIPage class is an abstract base for creating server-driven user interfaces for a specific player. It functions as a server-side controller and state machine for a UI screen that is rendered on the client. This class does not contain rendering logic; instead, its primary responsibility is to generate a series of commands and event bindings that describe the UI's structure, state, and interactivity.

This component acts as a bridge between high-level server game logic and the low-level UI protocol. Concrete implementations define a specific UI screen (e.g., a shop menu, a team selector, a quest journal) by implementing the abstract **build** method.

The system relies on two key builder objects:
*   **UICommandBuilder:** Assembles a list of drawing and layout commands (e.g., add panel, set text, position button).
*   **UIEventBuilder:** Binds client-side UI events (e.g., button click, text input) to server-side data handlers.

When a page is built or updated, these builders generate a payload that is packaged into a **CustomPage** network packet and sent to the player's client. The client then interprets this payload to render and manage the UI. The class name of the concrete subclass is used as the unique identifier for the page in the network protocol, ensuring the client and server are synchronized.

### Lifecycle & Ownership

-   **Creation:** A CustomUIPage instance is created by server-side game logic when a player needs to be shown a dynamic UI. This is typically triggered by a player action, such as interacting with an NPC, using an item, or executing a command. The newly created instance is then passed to the player's **PageManager** to be displayed.
-   **Scope:** The object's lifetime is directly tied to the visibility of the UI on the player's screen. It persists as long as the player has that specific page open. The **CustomPageLifetime** enum provides fine-grained control over its persistence, dictating whether it should close upon player death, respawn, or other game events.
-   **Destruction:** The instance is marked for garbage collection when the player's PageManager switches to a different page or closes the UI entirely. The **onDismiss** method is invoked as a final callback, allowing for state cleanup or other logic to be executed just before the page becomes inactive.

## Internal State & Concurrency

-   **State:** This class is inherently stateful and mutable. It holds a direct reference to the player entity (**PlayerRef**) and its configured lifetime. Concrete subclasses are expected to add their own state fields to manage the UI's data (e.g., a list of items for sale, the current page number in a book).
-   **Thread Safety:** **This class is not thread-safe.** All methods on a CustomUIPage instance must be invoked from the main server thread (the world tick thread). The design relies heavily on accessing the single-threaded **EntityStore** via the PlayerRef. Any attempt to call its methods from an asynchronous task or worker thread will result in data corruption, race conditions, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, cmdBuilder, evtBuilder, store) | abstract void | O(N) | **Core Method.** Subclasses must implement this to define the UI. Populates the builders with commands and event bindings. N is the number of UI elements. |
| handleDataEvent(ref, store, rawData) | void | O(M) | Handles incoming data events from the client UI. Throws UnsupportedOperationException by default. M is the complexity of the event handling logic. |
| onDismiss(ref, store) | void | O(1) | Callback hook invoked when the page is closed or replaced. Intended for state cleanup. |
| setLifetime(lifetime) | void | O(1) | Updates the persistence rule for this UI page. |

## Integration Patterns

### Standard Usage

A concrete implementation must be created, and its instance must be passed to the player's PageManager to be displayed. The **build** method is where all UI definition logic resides.

```java
// In some server-side logic, e.g., an NPC interaction handler

// 1. Create an instance of your custom UI page for the target player
ShopUIPage shopUI = new ShopUIPage(playerRef, CustomPageLifetime.CLOSE_ON_DEATH);

// 2. Retrieve the player component from the entity store
Player playerComponent = store.getComponent(playerRef, Player.getComponentType());

// 3. Set the player's active page, which triggers the initial build and network packet
playerComponent.getPageManager().setPage(playerRef, store, shopUI);
```

### Anti-Patterns (Do NOT do this)

-   **Asynchronous Updates:** Do not call **rebuild** or **sendUpdate** from a separate thread. All UI modifications must be synchronized with the main server tick to prevent race conditions with the EntityStore.
-   **Caching EntityStore References:** Do not store the **Ref** or **Store** objects passed into **build** or **handleDataEvent** as class members. These objects can become stale between ticks. Always re-acquire them from the **playerRef** field when needed, as demonstrated by the internal helper methods.
-   **Ignoring onDismiss:** For UIs that manage temporary server state (e.g., locking an item for a transaction), failing to implement cleanup logic in **onDismiss** can lead to state leaks when the player closes the UI unexpectedly.

## Data Pipeline

The flow of data for displaying and interacting with a CustomUIPage follows a clear, server-authoritative path.

> **Initial Display Flow:**
> Server Logic -> `new CustomUIPage()` -> `PageManager.setPage()` -> **CustomUIPage.build()** -> `UICommandBuilder` -> `CustomPage` Packet -> Network Layer -> Client Renders UI

> **Client Interaction Flow:**
> Client UI Event (e.g., Button Click) -> Network Packet -> Server Network Handler -> **CustomUIPage.handleDataEvent()** -> Server Logic -> `CustomUIPage.rebuild()` or `sendUpdate()` -> `CustomPage` Packet -> Client Updates UI


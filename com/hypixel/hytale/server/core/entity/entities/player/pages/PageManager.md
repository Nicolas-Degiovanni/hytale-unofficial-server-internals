---
description: Architectural reference for PageManager
---

# PageManager

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages
**Type:** Stateful Component

## Definition
```java
// Signature
public class PageManager {
```

## Architecture & Concepts
The PageManager is a server-side component responsible for controlling the primary user interface view, or *Page*, for a single connected player. It acts as the authoritative controller for what the player sees, managing both standard game menus (like inventory) and fully custom, server-defined user interfaces.

This class serves as a critical bridge between server-side game logic and the client's UI rendering engine. Its primary function is to serialize UI state and commands into network packets (SetPage, CustomPage) and to process inbound UI events from the client (CustomPageEvent).

It maintains a tight integration with the WindowManager to orchestrate complex UI layouts that involve a base page overlaid with one or more windows. The PageManager is fundamentally a state machine, tracking the currently active page and managing its lifecycle, especially for dynamic CustomUIPage instances.

A key architectural feature is the acknowledgement system. When the server sends a new custom page or an update, it requires an explicit acknowledgement from the client before processing further data events. This ensures that the server and client UI states remain synchronized, preventing race conditions where the client might send data related to a UI that is already stale from the server's perspective.

## Lifecycle & Ownership
- **Creation:** An instance of PageManager is created as part of the component set for a server-side Player entity. It is instantiated alongside other player-centric managers during the player's initial construction.
- **Scope:** The lifecycle of a PageManager is directly bound to the player's session. It persists as long as the associated Player entity exists in the game world.
- **Destruction:** The object is eligible for garbage collection when the owning Player entity is destroyed, typically when the player disconnects from the server. There is no explicit destruction or cleanup method; it relies on standard Java garbage collection. The **init** method is called shortly after construction to inject dependencies like the PlayerRef and WindowManager.

## Internal State & Concurrency
- **State:** The PageManager is highly stateful and mutable. Its core state includes:
    - A reference to the owning player (PlayerRef).
    - A reference to the associated WindowManager.
    - A nullable reference to the currently active CustomUIPage instance.
    - An atomic counter, **customPageRequiredAcknowledgments**, which tracks the number of pending UI state updates that the client must acknowledge.

- **Thread Safety:** This class is **not** thread-safe, with one critical exception. The acknowledgement counter uses an AtomicInteger, making its increment and decrement operations safe to call from different threads. This is by design:
    - UI-mutating methods like **openCustomPage** are expected to be called from the main server game-tick thread.
    - The **handleEvent** method, particularly the Acknowledge case, is called by the network layer, which may operate on a separate thread.
    
    **WARNING:** All methods other than the internal atomic operations on **customPageRequiredAcknowledgments** must be synchronized externally or, as intended, called exclusively from the main game thread to prevent state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(playerRef, windowManager) | void | O(1) | Initializes the manager with player and window dependencies. Must be called before any other method. |
| setPage(ref, store, page) | void | O(1) | Replaces the player's current UI with a standard, predefined Page. Dismisses any active custom page. |
| openCustomPage(ref, store, page) | void | O(N) | Builds and displays a new server-defined custom UI. The complexity depends on the UI build logic within the CustomUIPage. |
| setPageWithWindows(...) | boolean | O(N) | Atomically sets a standard page and opens a set of windows. Returns false if the operation fails. |
| openCustomPageWithWindows(...) | boolean | O(N) | Atomically opens a custom page and a set of windows. Returns false if the operation fails. |
| handleEvent(ref, store, event) | void | O(1) | The entry point for processing UI events sent from the client, such as dismissals, data submissions, or acknowledgements. |
| clearCustomPageAcknowledgements() | void | O(1) | Forcibly resets the acknowledgement counter to zero. Use with extreme caution as it can de-synchronize client and server state. |

## Integration Patterns

### Standard Usage
The PageManager should be retrieved from a Player entity's component system. Game logic then calls its methods to manipulate the player's UI state in response to game events.

```java
// Example: Opening a custom welcome screen for a player
Player player = getPlayerFromContext();
PageManager pageManager = player.getPageManager();
CustomUIPage welcomeScreen = new WelcomeScreenUI("Welcome, " + player.getName());

// The openCustomPage method handles building the UI, sending the packet,
// and setting the internal state to expect events from this new page.
pageManager.openCustomPage(player.getRef(), player.getStore(), welcomeScreen);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use **new PageManager()**. The manager is tightly coupled to a player's lifecycle and must be created and managed by the core entity system.
- **Skipping Initialization:** Calling any method before **init** has been successfully invoked will result in a NullPointerException.
- **State Mutation During Acknowledgement:** Do not attempt to send a new custom page while the **customPageRequiredAcknowledgments** counter is greater than zero. The system is designed to block data events during this period, but sending a new page can lead to unpredictable client-side behavior.
- **Ignoring Failure Cases:** The methods **setPageWithWindows** and **openCustomPageWithWindows** can return false. Code must handle this failure, as it indicates the player's UI was not updated as requested.

## Data Pipeline

The PageManager facilitates a bidirectional data flow between the server's game logic and the player's client.

> **Outbound Flow (Server to Client):**
> Game Logic -> **PageManager.openCustomPage()** -> CustomUIPage.build() -> UICommandBuilder -> CustomPage Packet -> PlayerPacketHandler -> Network Layer -> Client UI Engine

> **Inbound Flow (Client to Server):**
> Client UI Interaction -> CustomPageEvent Packet -> Network Layer -> Server Packet Processor -> **PageManager.handleEvent()** -> CustomUIPage.handleDataEvent() -> Game Logic Update


---
description: Architectural reference for ObjectiveAdminPanelPage
---

# ObjectiveAdminPanelPage

**Package:** com.hypixel.hytale.builtin.adventure.objectives.admin
**Type:** Transient

## Definition
```java
// Signature
public class ObjectiveAdminPanelPage extends BasicCustomUIPage {
```

## Architecture & Concepts
The ObjectiveAdminPanelPage is a server-side representation of a dynamic user interface component. It functions as a *View Model* within the server's UI framework. Its primary responsibility is to translate raw game state—specifically, the status of all registered objectives—into a series of declarative commands that the client can render.

This class acts as a bridge between the core objective system and the player's client. It queries global data sources like the ObjectiveDataStore and the Universe, formats this data, and then uses a UICommandBuilder to construct a payload for the client. This pattern decouples the server's complex game logic from the client's rendering concerns, allowing the UI layout (defined in .ui files) to be modified independently of the data-providing code.

This component is designed exclusively for administrative purposes, providing a real-time view of objective progress across all players on the server.

### Lifecycle & Ownership
- **Creation:** An instance of ObjectiveAdminPanelPage is created on-demand by a server-side system, typically in response to an administrative command or action initiated by a specific player. The constructor requires a PlayerRef, binding the page instance to a single player's UI context.
- **Scope:** The object's lifetime is extremely short. It exists only for the duration of the UI generation process. Once its build method has been called by the UI framework and the resulting commands have been sent to the client, the server-side object has served its purpose. The CustomPageLifetime.CanDismiss flag passed to the superclass constructor further signals its ephemeral nature to the UI system.
- **Destruction:** The object is eligible for garbage collection immediately after the transaction that displays the UI to the player is complete. There are no persistent references to it on the server.

## Internal State & Concurrency
- **State:** This class is effectively stateless. While it holds a reference to a PlayerRef, this is an immutable identifier. All data displayed in the UI is fetched from external, authoritative sources (ObjectivePlugin, Universe) during the execution of the build method. It does not cache or store any objective or player data.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, used, and discarded within the confines of a single, player-specific execution context, such as a server game tick or a command handler thread. It relies on the thread safety of the global singletons it accesses, such as Universe.get().

## API Surface
The public API is minimal, as the class is primarily controlled by the server's UI framework through its parent class contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(UICommandBuilder) | void | O(N * M) | Populates the provided UICommandBuilder with commands to construct and populate the admin panel. Complexity is O(N * M) where N is the total number of objectives and M is the number of players associated with an objective. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly. The server's UI framework instantiates and manages it. A typical invocation would be triggered by a command handler pushing the page to a player's UI stack.

```java
// Example of a hypothetical command handler
// This code would exist outside of the ObjectiveAdminPanelPage class.

PlayerRef adminPlayer = ...; // Get the player who ran the command
PageManager pageManager = adminPlayer.getPageManager();

// The framework creates and builds the page internally
pageManager.push(new ObjectiveAdminPanelPage(adminPlayer));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Building:** Do not create an instance and call the build method yourself. The UI framework is responsible for providing the correct UICommandBuilder and managing the page's lifecycle.
- **State Caching:** Do not modify this class to cache objective data. It is designed to reflect the absolute latest state from the data store every time it is displayed. Caching would introduce stale data.
- **Cross-Player Reuse:** Never create a single instance and attempt to show it to multiple players. The page is fundamentally bound to the PlayerRef provided during construction.

## Data Pipeline
The flow of data for rendering this panel is unidirectional, originating from server-side game state and terminating on the client's screen.

> Flow:
> Admin Player Input (e.g., command) -> Server Command Handler -> **ObjectiveAdminPanelPage.build()** -> UICommandBuilder -> Network Packet -> Client UI Renderer


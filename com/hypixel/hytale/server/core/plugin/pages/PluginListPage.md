---
description: Architectural reference for PluginListPage
---

# PluginListPage

**Package:** com.hypixel.hytale.server.core.plugin.pages
**Type:** Transient

## Definition
```java
// Signature
public class PluginListPage extends InteractiveCustomUIPage<PluginListPage.PluginListPageEventData> {
```

## Architecture & Concepts
The PluginListPage class is a server-side controller that manages the user interface for listing and toggling server plugins. It does not represent the UI elements directly but rather orchestrates their creation, state, and event handling on the client through a command-based protocol. This class acts as a bridge between a specific player's UI session and the server's global PluginManager and CommandManager singletons.

Its core responsibility is to translate the state of available server plugins into a series of UI commands that are sent to the player's client. When the player interacts with the UI (e.g., clicking a button, toggling a checkbox), the client sends a structured event payload back to the server, which is then handled by this object's `handleDataEvent` method. Instead of modifying plugin state directly, it delegates these actions by executing server commands, ensuring that all plugin state changes follow the standard, permission-checked command pipeline.

This class is fundamentally a stateful, player-specific component. Each player who opens the plugin list UI will have a unique instance of PluginListPage created for them.

### Lifecycle & Ownership
- **Creation:** An instance of PluginListPage is created by the server's UI framework when a specific player is shown this page. The constructor requires a PlayerRef, tightly coupling the instance to a single player's context.
- **Scope:** The object's lifetime is strictly bound to the time the player has the plugin list UI open. It persists across multiple interactions within that single viewing session.
- **Destruction:** The `onDismiss` method is invoked when the player closes the UI. This method triggers cleanup, primarily by calling `deregisterPluginListPage` on the PluginListPageManager. Once deregistered and no other references exist, the object is eligible for garbage collection.

## Internal State & Concurrency
- **State:** This class is highly **mutable**. It maintains a significant amount of state specific to the player's UI session, including:
    - A cached list of all available plugins (`availablePlugins`).
    - A filtered list of plugins currently visible to the player (`visiblePlugins`), which changes based on UI filters.
    - The currently selected plugin in the details view (`selectedPlugin`).
    - A reference to the player's specific UI settings (`playerSessionSettings`).
    This state is constructed during the `build` method and is modified in response to player-initiated events.

- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread (the game tick). All methods operate under the assumption of single-threaded access within the server's update loop. The use of `Ref<EntityStore>` and `Store<EntityStore>` as method parameters are strong indicators of this design.

    **Warning:** Concurrent modification from other threads will lead to race conditions, inconsistent UI state, and server instability. Do not share instances of this class across threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(...) | void | O(N) | Constructs the initial UI state. Populates the plugin list by querying PluginManager. Complexity is proportional to the number of available plugins. |
| handleDataEvent(...) | void | O(N) | Processes an incoming event from the client. Dispatches actions like selecting a plugin or toggling its state. May trigger a full UI rebuild. |
| onDismiss(...) | void | O(1) | Called when the player closes the UI. Performs necessary cleanup and deregistration. |
| handlePluginChangeEvent(...) | void | O(N) | An external callback to update the UI when a plugin's state changes outside of this page's direct control. Finds the relevant plugin in its cached list to update the checkbox. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation by developers. It is managed entirely by the server's core UI and plugin management systems. A system like PluginListPageManager may hold a reference to active instances to push external state changes, such as a plugin being enabled or disabled by a server administrator.

```java
// Example of an external system notifying the page of a change
// This code would exist within a manager class, not typical user code.

PluginListPage activePage = getActivePageForPlayer(playerRef);
if (activePage != null) {
    // Notify the page that the 'com.example.myplugin' state changed
    activePage.handlePluginChangeEvent(pluginIdentifier, newState);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PluginListPage(playerRef)`. The lifecycle is managed by the server's UI framework. Manual creation will result in a disconnected object that does not receive events or properly render for the player.
- **State Caching:** Do not cache a reference to a PluginListPage instance beyond its lifecycle. Once `onDismiss` is called, the object should be considered invalid and will not be updated.
- **Cross-Thread Access:** Never invoke methods on this class from any thread other than the main server thread. This will corrupt its internal state and violate the server's threading model.

## Data Pipeline
The flow of data for this component is bidirectional, handling both rendering commands sent to the client and interaction events received from the client.

> **Outbound Flow (Server to Client):**
> PluginManager State -> **PluginListPage.build()** -> UICommandBuilder -> Protocol Encoder -> Network Packet -> Client UI Renderer

> **Inbound Flow (Client to Server):**
> Player Interaction (Click) -> Client UI Event -> Network Packet (with PluginListPageEventData) -> Protocol Decoder -> **PluginListPage.handleDataEvent()** -> CommandManager -> Plugin State Change


---
description: Architectural reference for ConfigureInstanceBlockPage
---

# ConfigureInstanceBlockPage

**Package:** com.hypixel.hytale.builtin.instances.page
**Type:** Transient

## Definition
```java
// Signature
public class ConfigureInstanceBlockPage extends InteractiveCustomUIPage<ConfigureInstanceBlockPage.PageData> {
```

## Architecture & Concepts
The **ConfigureInstanceBlockPage** class is a server-side UI controller that manages the configuration screen for a **ConfigurableInstanceBlock**. It operates within the server's Entity Component System (ECS) and custom UI framework, acting as the bridge between a player's client-side UI interactions and the server's world state.

This class embodies a server-authoritative UI pattern. The server defines the UI layout (via `Pages/ConfigureInstanceBlockPage.ui`), populates it with current data, binds events, and processes the results. The client is responsible for rendering and forwarding input, but all logic and state changes are handled exclusively by this class on the server.

Its primary responsibilities are:
1.  **Building the UI:** Dynamically constructing the UI commands needed to render the configuration page, including populating dropdowns with available world instances.
2.  **Event Handling:** Processing data events sent from the client when the player interacts with UI elements like text fields, checkboxes, and buttons.
3.  **State Management:** Modifying the state of the target **ConfigurableInstanceBlock** component based on player input and persisting those changes by marking the underlying world chunk for saving.

## Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by server logic, typically in response to a player interacting with a **ConfigurableInstanceBlock** in the game world. It is instantiated directly with a reference to the player (**PlayerRef**) and the target block entity (**Ref<ChunkStore>**).

-   **Scope:** The object's lifetime is ephemeral, strictly tied to the duration of the player's interaction with the configuration UI. It persists only as long as it is the active page in the player's **PageManager**.

-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the player's **PageManager** transitions to a new page (e.g., **Page.None** after saving) or when the player disconnects. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains a direct reference to the **ConfigurableInstanceBlock** component it is editing. Furthermore, it holds its own temporary copies of configuration values like **positionOffset** and **rotation**. This local state is used to manage the UI's appearance (e.g., showing/hiding input fields) without committing changes to the underlying block component until the player explicitly saves.

-   **Thread Safety:** **ConfigureInstanceBlockPage** is not thread-safe. All of its methods are designed to be called from the server's main game thread, which has synchronous access to the world state. Accessing an instance from any other thread will result in race conditions, data corruption, and likely crash the server. All interactions must be marshaled through the server's task scheduler or event system.

## API Surface
The public contract is defined by its inheritance from **InteractiveCustomUIPage**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(...) | void | O(N) | Constructs the UI state by issuing commands to a **UICommandBuilder**. Complexity is proportional to the number of UI elements. |
| handleDataEvent(...) | void | O(1) | The core logic callback. Processes deserialized **PageData** from the client to update internal state or the world state. |

## Integration Patterns

### Standard Usage
This class is intended to be used by server-side game logic, such as a block interaction handler. The handler creates the page and assigns it to the player, transferring UI control.

```java
// Example from a hypothetical block interaction handler
Player player = ...;
Ref<ChunkStore> blockEntityRef = ...;

// Create a new page instance for this specific interaction
ConfigureInstanceBlockPage page = new ConfigureInstanceBlockPage(player.getRef(), blockEntityRef);

// Set this as the player's active UI page
player.getPageManager().setPage(player.getRef(), player.getStore(), page);
```

### Anti-Patterns (Do NOT do this)
-   **Caching or Reusing Instances:** Do not cache instances of this class. Each instance is tied to a specific block and a specific UI session. It must be created new for each interaction.

-   **External State Modification:** Do not modify the internal **positionOffset** or **rotation** fields from outside the class. These are managed internally in response to UI events to drive partial UI updates.

-   **Invoking from Worker Threads:** Never call **build** or **handleDataEvent** from an asynchronous task or a different thread. All interactions must occur on the main server thread that owns the world state.

## Data Pipeline
The flow of data from client interaction to server state change is a complete round-trip, orchestrated by the server.

> Flow:
> Player UI Input -> Client sends UI Event Packet -> Server Network Layer -> **PageData** Deserialization -> **ConfigureInstanceBlockPage.handleDataEvent** -> **ConfigurableInstanceBlock** Component Update -> **BlockComponentChunk.markNeedsSaving()** -> World Save System


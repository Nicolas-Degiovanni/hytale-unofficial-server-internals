---
description: Architectural reference for EntitySpawnPage
---

# EntitySpawnPage

**Package:** com.hypixel.hytale.server.npc.pages
**Type:** Transient

## Definition
```java
// Signature
public class EntitySpawnPage extends InteractiveCustomUIPage<EntitySpawnPage.EntitySpawnPageEventData> {
```

## Architecture & Concepts
The EntitySpawnPage class is a server-side controller for a complex, interactive user interface designed for spawning entities in the game world. It functions as the state machine and logic handler for the "Entity Spawn" tool, typically used by administrators or in creative game modes.

Architecturally, this class acts as a bridge between the client-side UI and the server's core Entity Component System (ECS). It translates user input, received as `EntitySpawnPageEventData` objects, into concrete actions within the server's world state, such as creating preview entities, filtering asset lists, and ultimately spawning persistent entities.

It inherits from `InteractiveCustomUIPage`, indicating its role within the server's UI framework. This framework dictates a clear separation of concerns:
1.  **Layout Definition:** The `build` method defines the initial structure of the UI and establishes bindings between UI elements and server-side event handlers.
2.  **Event Handling:** The `handleDataEvent` method is the central hub for processing all interactions from the client, such as button clicks, slider adjustments, and text input.
3.  **State Management:** The class maintains the complete state of the UI, including the active tab (NPCs, Items, Models), search queries, selected assets, and parameters for the entity to be spawned (rotation, scale).
4.  **Visual Feedback:** A key feature is the management of a temporary "preview" entity. This is a non-persistent, client-visible entity spawned in the world to show the user exactly what they are about to create and where. This provides immediate visual feedback for position, rotation, and scale adjustments.

## Lifecycle & Ownership
- **Creation:** An instance of EntitySpawnPage is created on-demand when a player is directed to this specific UI. This is typically managed by a `PageManager` associated with the player entity, which would instantiate the class and set it as the player's active page.
- **Scope:** The object's lifetime is strictly tied to the player's interaction with the entity spawn UI. It persists only as long as the page is visible to the player.
- **Destruction:** The instance is marked for garbage collection when the player closes the UI, either by dismissing it or navigating to another page. The `onDismiss` method is the critical finalization hook, responsible for cleaning up any world-state resources, most importantly destroying the preview entity to prevent it from remaining orphaned in the world.

## Internal State & Concurrency
- **State:** This class is highly **mutable** and stateful. It maintains numerous fields representing the UI's current condition, such as `activeTab`, `searchQuery`, `selectedNpcRole`, `currentScale`, and a reference to the preview entity (`modelPreview`). This state is modified transactionally within the `handleDataEvent` method in response to user actions.

- **Thread Safety:** **This class is not thread-safe and must only be accessed from the main server thread.** Like most game engine components that interact with the world state, all of its methods are designed to execute synchronously within the server's single-threaded game loop. Any concurrent access would lead to severe race conditions, data corruption, and unpredictable server crashes. The UI framework guarantees that all event handling methods are called on the correct thread.

## API Surface
The public contract is defined by its role as an `InteractiveCustomUIPage`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Invoked once by the UI framework to construct the initial UI. Defines layout and binds events. Complexity is proportional to the number of initial UI elements. |
| handleDataEvent(ref, store, data) | void | O(M log M) | The primary event handler. Processes user input from the client. Complexity is dominated by asset filtering and sorting operations. |
| onDismiss(ref, store) | void | O(1) | Invoked by the UI framework when the page is closed. Responsible for resource cleanup, primarily the preview entity. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by most plugin or gameplay code. It is instantiated and managed by the server's core UI system, the `PageManager`. A developer would typically open this UI for a player as follows:

```java
// Correctly open the page for a player
Player player = store.getComponent(playerRef, Player.getComponentType());
player.getPageManager().setPage(playerRef, store, new EntitySpawnPage(playerRef));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Management:** Do not create an instance of `EntitySpawnPage` and attempt to call its methods manually. The class relies on the `PageManager` to correctly orchestrate its lifecycle and event flow.
- **State Tampering:** Do not retain a reference to an `EntitySpawnPage` instance and modify its state from outside the `handleDataEvent` flow. This will desynchronize the server state from the client UI.
- **Ignoring `onDismiss`:** Modifying this class to remove the call to `clearPreview` in `onDismiss` would introduce a critical bug, leaving orphaned preview entities scattered across the world.

## Data Pipeline
The flow of information for a single user interaction, such as changing the scale slider, follows a well-defined path between the client and server.

> Flow:
> Client UI Interaction (Slider Drag) -> Client sends `CustomUIEvent` Packet -> Server Network Layer -> `PageManager` routes event -> **EntitySpawnPage.handleDataEvent** -> Internal state (`currentScale`) is updated -> Preview entity is updated in the world (`Store<EntityStore>`) -> `UICommandBuilder` is populated with changes -> `sendUpdate` sends UI update packet to Client -> Client UI Engine re-renders elements (e.g., scale text label).


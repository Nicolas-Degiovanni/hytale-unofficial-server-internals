---
description: Architectural reference for EntityUIModule
---

# EntityUIModule

**Package:** com.hypixel.hytale.server.core.modules.entityui
**Type:** Singleton

## Definition
```java
// Signature
public class EntityUIModule extends JavaPlugin {
```

## Architecture & Concepts
The EntityUIModule is a foundational server-side plugin responsible for managing all user interface elements attached to game entities. This includes elements such as health bars, nameplates, status effect icons, and floating combat text. It operates as a bridge between static asset definitions and the dynamic, stateful world of the Entity Component System (ECS).

Architecturally, this module serves three primary functions:

1.  **Asset Registration:** It registers the various types of `EntityUIComponent` assets and their associated network codecs with the server's `AssetRegistry`. This allows the server to understand, load, and serialize UI definitions from game files.
2.  **Component Definition:** It introduces the `UIComponentList` component into the ECS. This component acts as a container on an entity, holding a list of all active UI elements that should be rendered for it.
3.  **System Registration:** It registers a suite of core ECS systems (`UIComponentSystems`) that contain the logic for managing the lifecycle of UI components. These systems are responsible for adding, updating, and removing UI elements from entities based on game state and player visibility.

The module's dependency on `EntityStatsModule` is critical; it implies that UI elements are not merely cosmetic but are data-driven displays directly reflecting an entity's underlying statistics, such as health or mana.

## Lifecycle & Ownership
-   **Creation:** The EntityUIModule is instantiated once by the server's core plugin loader during the bootstrap sequence. Its `MANIFEST` file declares its dependencies, ensuring it is loaded *after* the `EntityStatsModule` to guarantee system availability.
-   **Scope:** As a core singleton plugin, its lifecycle is bound to the server process itself. It persists for the entire duration of a server session.
-   **Destruction:** The module is destroyed only when the server shuts down. Cleanup is managed by the Java Virtual Machine; there is no explicit teardown logic within the class.

## Internal State & Concurrency
-   **State:** The module instance itself is largely stateless after initialization. Its primary state, the `uiComponentListType` handle, is assigned during the `setup` phase and is immutable thereafter. The true state managed by this module is decentralized, stored within the `UIComponentList` components attached to individual entities across all game worlds.

-   **Thread Safety:** The module's public API is thread-safe for read operations after the server's initial setup phase is complete. The systems it registers operate within the server's ECS, which dictates its own threading model. The `onLoadedAssetsEvent` handler demonstrates awareness of concurrency by using `forEachEntityParallel`, indicating that the underlying entity store is designed for safe, parallel iteration. Any logic executed by these systems or event handlers must adhere to the concurrency guarantees of the ECS framework.

## API Surface
The direct API surface is minimal by design, as developers primarily interact with the systems this module establishes through data (asset files) rather than code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | EntityUIModule | O(1) | Static accessor for the singleton instance. Throws NullPointerException if called before the plugin is loaded. |
| getUIComponentListType() | ComponentType | O(1) | Returns the unique type identifier for the UIComponentList, used for ECS queries. |

## Integration Patterns

### Standard Usage
Direct interaction with the EntityUIModule class is rare. The primary integration pattern is declarative, through the creation of asset files. A developer or designer defines an `EntityUIComponent` in a data file (e.g., JSON), and the systems registered by this module automatically apply that UI to the relevant entities in-game.

For programmatic interaction, other systems would query for entities possessing the `UIComponentList` component.
```java
// Example of another system interacting with the component
// defined by EntityUIModule.
ComponentType<EntityStore, UIComponentList> uiListType = EntityUIModule.get().getUIComponentListType();

// Find an entity and get its UI component list
UIComponentList uiList = entity.get(uiListType);
if (uiList != null) {
    // Modify the list to add or remove UI elements
    uiList.add(newCombatText("100"));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new EntityUIModule()`. The server's plugin loader is solely responsible for its creation. Attempting to do so manually will fail or destabilize the server.
-   **Premature Access:** Do not call `EntityUIModule.get()` from the constructor of another plugin. The load order is not guaranteed, and it may return null. Access the module within your plugin's `setup` or `enable` phase.
-   **Stateful Logic in Handlers:** The `onLoadedAssetsEvent` handler is executed in parallel across entities. Avoid introducing logic that relies on shared, mutable state without proper synchronization, as this will create race conditions.

## Data Pipeline
The module orchestrates the flow of UI data from configuration files to the player's screen.

> Flow:
> 1. **Asset File:** A designer creates a JSON or HOCON file defining an `EntityUIComponent`.
> 2. **AssetStore:** On server start or asset reload, the `HytaleAssetStore` parses this file into an `EntityUIComponent` Java object.
> 3. **LoadedAssetsEvent:** The asset system fires an event to signal that UI assets have been loaded.
> 4. **EntityUIModule Listener:** The `onLoadedAssetsEvent` method in this module receives the event.
> 5. **ECS Parallel Update:** The listener schedules a task to iterate over all entities in all worlds, calling the `update` method on their `UIComponentList` component. This applies the new asset definitions.
> 6. **UIComponentSystems.Update:** On every server tick, this ECS system processes entities with a `UIComponentList`.
> 7. **Packet Generation:** The system reads the component's state and, if the entity is tracked by a player, generates network packets describing the UI elements and their current state (e.g., health bar percentage).
> 8. **Network Layer:** Packets are dispatched to all relevant game clients.
> 9. **Client-Side Rendering:** The client receives the packets and renders the corresponding UI elements in the world.


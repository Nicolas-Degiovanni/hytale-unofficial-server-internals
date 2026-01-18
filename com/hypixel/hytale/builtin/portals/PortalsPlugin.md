---
description: Architectural reference for PortalsPlugin
---

# PortalsPlugin

**Package:** com.hypixel.hytale.builtin.portals
**Type:** Singleton

## Definition
```java
// Signature
public class PortalsPlugin extends JavaPlugin {
```

## Architecture & Concepts
The PortalsPlugin serves as the central bootstrap and registry for the entire Portals gameplay feature. As a subclass of JavaPlugin, its primary responsibility is to integrate all portal-related components, systems, commands, and configurations into the core server engine during the startup sequence.

It does not participate in the main game loop directly. Instead, its one-time `setup` method acts as a master configuration script that:
1.  **Registers ECS Primitives:** Defines and registers core data structures like the `PortalWorld` resource, `PortalDevice` component, and `VoidEvent` component. This makes the engine aware of how to store and serialize portal-related data.
2.  **Initializes Logic Systems:** Instantiates and registers all systems that govern portal behavior, such as `PortalInvalidDestinationSystem` for handling broken links and `VoidEventStagesSystem` for managing void invasions.
3.  **Binds Event Listeners:** Subscribes to global engine events, most notably `AddWorldEvent` and `RemoveWorldEvent`. This allows the portal feature to react to the creation and destruction of worlds, a critical function for maintaining portal link integrity.
4.  **Exposes Commands & Interactions:** Registers player-facing commands (e.g., `LeaveCommand`) and custom interaction types (e.g., `EnterPortalInteraction`), making the feature accessible to players and content creators.

Architecturally, this class is the glue that connects dozens of disparate classes into a cohesive feature. It establishes the foundational rules and data types upon which all other portal logic is built. The static constant `MAX_CONCURRENT_FRAGMENTS` also defines a key architectural constraint for the entire system.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the server's plugin loader during the server boot process. An initialization context, `JavaPluginInit`, is provided by the loader, granting access to the engine's core registries.
-   **Scope:** The PortalsPlugin is a global singleton that persists for the entire server session. The static `instance` field is set within the `setup` method, providing a globally accessible reference point.
-   **Destruction:** The object is destroyed only when the server shuts down or the plugin is unloaded. There is no explicit teardown logic; cleanup is managed by the Java Virtual Machine and the core plugin host.

## Internal State & Concurrency
-   **State:** The internal state consists of references to registered `ComponentType` and `ResourceType` objects. These references are assigned once during the `setup` method and are immutable thereafter. The class does not hold or manage dynamic game state; that responsibility is delegated to the components and systems it registers.
-   **Thread Safety:** This class is designed to be thread-safe.
    -   The `setup` method is guaranteed by the engine to be called only once from the main server thread during initialization.
    -   The public getters return immutable references assigned during setup, which is a safe operation.
    -   The `turnOffPortalWhenWorldRemoved` event listener correctly dispatches work to the appropriate world's thread via `world.execute`, preventing concurrency issues when modifying world state.
    -   The `countActiveFragments` method iterates a collection of worlds, which is assumed to be a thread-safe operation provided by the `Universe` service.

## API Surface
The public API is minimal, primarily exposing the singleton instance and type identifiers for use in other systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInstance() | PortalsPlugin | O(1) | **STATIC**. Retrieves the singleton instance of the plugin. Throws NullPointerException if accessed before the plugin is initialized. |
| countActiveFragments() | int | O(N) | Counts the number of active worlds that are configured as portal "fragments". N is the total number of worlds. |
| getPortalResourceType() | ResourceType | O(1) | Returns the registered type handle for the PortalWorld resource. |
| getPortalDeviceComponentType() | ComponentType | O(1) | Returns the registered type handle for the PortalDevice component. |
| getVoidEventComponentType() | ComponentType | O(1) | Returns the registered type handle for the VoidEvent component. |
| getVoidPortalComponentType() | ComponentType | O(1) | Returns the registered type handle for the VoidSpawner component. |

## Integration Patterns

### Standard Usage
The plugin is almost always accessed via its static `getInstance` method to retrieve the registered type handles. These handles are then used with world or entity stores to query and manipulate portal data.

```java
// Example: Finding all PortalDevice components in a chunk
PortalsPlugin plugin = PortalsPlugin.getInstance();
ComponentType<ChunkStore, PortalDevice> deviceType = plugin.getPortalDeviceComponentType();

for (PortalDevice device : chunkStore.getComponents(deviceType)) {
    // ... logic to inspect the portal device
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PortalsPlugin()`. The plugin loader is the sole authority for its creation. Doing so will result in a non-functional, disconnected instance.
-   **Premature Access:** Do not call `PortalsPlugin.getInstance()` from another plugin's constructor or during a very early initialization phase. The static `instance` may not be set yet, resulting in a NullPointerException. Access should occur within a system's logic or after the server has fully started.
-   **State Modification:** Do not attempt to re-register systems or components after the `setup` phase has completed. The engine's registries are not designed for runtime modification and this will lead to undefined behavior.

## Data Pipeline
This class does not process a continuous stream of data. Instead, it wires together event-driven pipelines. The following demonstrates the flow of data and control when a portal world is removed, a critical process managed by this plugin.

> **Flow: Portal World Removal**
>
> 1.  **External Trigger:** A game event (e.g., a player breaking a portal device block) causes a system to request the removal of a portal world.
> 2.  **Engine Event:** The core `Universe` service processes the request and broadcasts a global `RemoveWorldEvent`.
> 3.  **Plugin Listener:** The `turnOffPortalWhenWorldRemoved` method in `PortalsPlugin` receives this event.
> 4.  **World Iteration:** The listener iterates through all *other* remaining worlds in the `Universe`.
> 5.  **Task Dispatch:** For each world, it schedules a task on that world's dedicated thread to execute `PortalInvalidDestinationSystem.turnOffPortalsInWorld`.
> 6.  **System Execution:** The `PortalInvalidDestinationSystem` runs, querying for all `PortalDevice` components in its world.
> 7.  **State Update:** It checks if any `PortalDevice` points to the now-removed world. If a match is found, the component's state is updated to "inactive", visually and functionally disabling the portal.


---
description: Architectural reference for InteractionModule
---

# InteractionModule

**Package:** com.hypixel.hytale.server.core.modules.interaction
**Type:** Singleton / Plugin

## Definition
```java
// Signature
public class InteractionModule extends JavaPlugin {
```

## Architecture & Concepts
The InteractionModule is a foundational server plugin that establishes the entire framework for player interactions with the game world. It acts as the central registry and dispatcher for all actions originating from player input, such as mouse clicks and movement.

Architecturally, this module serves three primary purposes:

1.  **Registration Hub:** During server initialization, its `setup` method is invoked. This method systematically registers dozens of concrete **Interaction** types (e.g., PlaceBlockInteraction, DamageEntityInteraction, LaunchProjectileInteraction) with the engine's core codec and asset registries. This makes the server aware of all possible actions that can be defined in asset files.
2.  **Component & System Provisioning:** It registers critical Entity Component System (ECS) components like **InteractionManager** and **ChainingInteraction.Data**, which are attached to entities to manage their interaction state. It also registers the core **InteractionSystems** that process these components each tick.
3.  **Input Packet Gateway:** It provides the `doMouseInteraction` method, which is the primary entry point for raw `MouseInteraction` network packets received from a client. This method translates low-level packet data into high-level, semantic game events like `PlayerMouseButtonEvent`, which are then dispatched to the server's event bus for other systems to consume.

In essence, this module is the bridge between the network layer and the game's action and combat systems. It decouples raw input from the complex, data-driven logic defined in game assets.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the server's plugin loader during the bootstrap sequence. The static `instance` field is set within the constructor, enforcing a singleton pattern for the lifetime of the server.
- **Scope:** The InteractionModule is a persistent, server-scoped singleton. It is initialized once when the server starts and lives until the server shuts down.
- **Destruction:** The object is eligible for garbage collection only upon server shutdown. There is no explicit destruction or cleanup logic within the class itself; this is managed by the JavaPlugin lifecycle.

## Internal State & Concurrency
- **State:** The module's state consists primarily of handles to registered component and resource types (e.g., `interactionManagerComponent`, `placedByComponentType`). This state is populated exclusively within the `setup` method during server initialization. After this initial phase, the state is effectively immutable.
- **Thread Safety:** This class is **not thread-safe**. The `setup` method must only be called by the main server thread during its initialization phase. The `doMouseInteraction` method is designed to be called from the server's main game loop or a dedicated network thread that has exclusive write access to the game state for a given tick.

**WARNING:** Invoking any method on this class from an arbitrary worker thread will lead to race conditions, state corruption, and server instability. All interaction with the game world must be synchronized with the main server tick.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | InteractionModule | O(1) | Retrieves the static singleton instance. |
| setup() | void | O(N) | **[Internal Engine Hook]** Registers all interaction assets, components, and systems. Catastrophic if called outside of server bootstrap. |
| doMouseInteraction(...) | void | O(log N) | Processes a raw mouse interaction packet. Involves entity lookups, component access, and event dispatching. |
| get...Component() | ComponentType | O(1) | Returns a handle to a registered ECS component type. |
| get...ResourceType() | ResourceType | O(1) | Returns a handle to a registered ECS resource type. |

## Integration Patterns

### Standard Usage
The InteractionModule is not intended for direct use by most game logic or other plugins. Its primary consumer is the server's network packet handler, which funnels player input into it. Other systems interact with the *results* of this module—namely the components, assets, and events it registers.

A typical developer will interact with the systems this module establishes, not the module itself. For example, they might create a new item asset that references a `RootInteraction` that was registered by this module.

```java
// This is a conceptual example of how the ENGINE uses the module.
// Developers typically do not call this method directly.

// Inside a hypothetical network packet handler...
InteractionModule interactionModule = InteractionModule.get();
Player player = ...; // The player associated with the packet
MouseInteraction packet = ...; // The inbound packet

// The engine dispatches the raw packet to the module for processing.
interactionModule.doMouseInteraction(player.getRef(), accessor, packet, player, playerRef);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new InteractionModule()`. The server's plugin loader is solely responsible for its creation. Use the static `get()` method if access to the instance is absolutely required.
- **Manual Setup:** Never call the `setup()` method. Doing so will cause duplicate registration errors and will corrupt the server's core registries.
- **Asynchronous Processing:** Do not call `doMouseInteraction` from a separate thread without synchronizing with the main game loop. This will bypass all engine-level thread safety mechanisms for world and entity modification.

## Data Pipeline
The module serves as a critical transformation step in the player input data pipeline. It converts raw, low-level network data into structured, high-level game events that the rest of the engine can understand and react to.

> Flow:
> Client Input -> `MouseInteraction` Network Packet -> Server Network Handler -> **InteractionModule.doMouseInteraction** -> `PlayerMouseButtonEvent` -> Server Event Bus -> InteractionSystems & Other Listeners


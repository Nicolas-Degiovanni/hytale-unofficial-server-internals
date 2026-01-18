---
description: Architectural reference for OpenCustomUIInteraction
---

# OpenCustomUIInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration Object

## Definition
```java
// Signature
public class OpenCustomUIInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts

The OpenCustomUIInteraction class is a data-driven action within the server's core Interaction System. It represents a specific, configured behavior: opening a custom user interface for a player. It is not a service or manager, but rather a stateless object that encapsulates the logic for a single type of interaction, typically defined in external configuration files like JSON.

The architectural foundation of this class relies on a combination of the **Strategy** and **Registry** patterns to achieve high extensibility.

1.  **Strategy Pattern:** The core logic of creating a specific UI page is delegated to a `CustomPageSupplier` instance. This decouples the generic action of "opening a UI" from the concrete implementation of *what* UI to create. Each instance of OpenCustomUIInteraction holds a reference to a specific supplier strategy.

2.  **Registry Pattern:** The static `PAGE_CODEC` field acts as a central registry. It maps string identifiers (used in configuration files) to the codecs responsible for creating different `CustomPageSupplier` strategies. This allows plugins and other modules to register new, custom UI behaviors that can be dynamically referenced by game content designers without modifying the core interaction code.

This design makes the interaction system extremely flexible. Content designers can define complex UI interactions declaratively in data files, while programmers can provide the underlying UI creation logic by registering new suppliers during server initialization.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly using the `new` keyword. They are instantiated by the engine's `Codec` deserialization system during server startup. The engine reads interaction definitions from game data files and uses the static `CODEC` field to construct and configure each OpenCustomUIInteraction object.
- **Scope:** An instance persists for the entire server session. As a stateless configuration object, a single instance can be reused for every corresponding interaction that occurs in the game world.
- **Destruction:** The object has no explicit destruction method. It is garbage collected when the server shuts down or when its corresponding interaction definition is unloaded from memory.

## Internal State & Concurrency
- **State:** An instance of OpenCustomUIInteraction is effectively **immutable** after its initial creation by the codec system. Its single state field, `customPageSupplier`, is set during deserialization and is not modified thereafter. The class holds no runtime state related to players or specific interaction events.

- **Thread Safety:** This class is **thread-safe**. Its immutable nature allows it to be safely accessed by multiple server threads. All mutable state changes (e.g., updating the player's open UI page) are managed through the `CommandBuffer` provided in the `InteractionContext`. The CommandBuffer system ensures that all modifications are queued and executed safely within the main server tick loop, preventing race conditions.

## API Surface
The primary public contract of this class is its static registration interface, used by plugins to extend its functionality. The `firstRun` method is an engine-level callback and should not be invoked directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerCustomPageSupplier(plugin, tClass, id, supplier) | static void | O(1) | Registers a new type of UI page supplier. This is the primary extension point. |
| registerSimple(plugin, tClass, id, supplier) | static void | O(1) | A convenience wrapper for registering a simple supplier that only depends on the player. |
| registerBlockCustomPage(...) | static void | O(1) | **Deprecated.** Registers a supplier for a UI associated with a block's `BlockState`. |
| registerBlockEntityCustomPage(...) | static void | O(1) | Registers a supplier for a UI associated with a block entity, such as a chest or furnace. |

## Integration Patterns

### Standard Usage
The primary interaction with this class is not to call its methods, but to register a new UI creation strategy during your plugin's initialization phase. This makes your custom UI available to be used in game data files.

```java
// In your plugin's onEnable() method:

// 1. Define the logic for creating your custom UI page.
//    This example creates a simple page that depends only on the player.
Function<PlayerRef, CustomUIPage> myPageSupplier = (playerRef) -> {
    return new MyCustomInfoPage(playerRef);
};

// 2. Register this logic with the interaction system under a unique ID.
OpenCustomUIInteraction.registerSimple(
    this, // Your PluginBase instance
    MyCustomInfoPage.class,
    "my_plugin:my_info_page", // The ID used in config files
    myPageSupplier
);

// Now, a content designer can create an interaction in a JSON file that
// uses "my_plugin:my_info_page" to open your UI.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new OpenCustomUIInteraction()`. The class is designed to be constructed by the engine's deserialization system. Manual creation will result in a non-functional object that is not part of the interaction system.
- **Calling firstRun:** Never call the `firstRun` method directly. It is a callback executed by the Interaction System, which provides a valid and safe `InteractionContext`. Manual invocation will bypass critical systems like cooldowns, permissions, and state management, likely causing server instability.
- **Runtime Registration:** Do not call registration methods after the server has finished its initialization phase. The codec registries are not designed for concurrent modification while the server is running and processing interactions. All registrations must occur during plugin loading.

## Data Pipeline
The flow of data and control for this interaction begins with a game data file and ends with a UI update on the client.

> Flow:
> Game Configuration File -> Engine Codec Deserialization -> Interaction System Registry -> Player Action (e.g., Right-Click) -> **OpenCustomUIInteraction.firstRun()** -> CustomPageSupplier Strategy -> PageManager -> Player Component State Change -> Server Network Tick -> Client UI Update


---
description: Architectural reference for OpenPageInteraction
---

# OpenPageInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration Object

## Definition
```java
// Signature
public class OpenPageInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts

The OpenPageInteraction class is a data-driven, server-side action that instructs a player's client to open a specific user interface page. It is a concrete implementation within the server's broader **Interaction System**, which handles all forms of entity-to-entity and entity-to-world interactions.

Architecturally, this class is not a service or manager but a lightweight, serializable **Configuration Object**. Its primary role is to be defined within external data files (e.g., JSON) and deserialized at runtime by the engine's `Codec` system. This pattern decouples game logic from game content, allowing designers to specify UI-opening behaviors on blocks, items, or NPCs without writing new code.

The class operates within the server's Entity Component System (ECS). When an interaction triggers this action, its `firstRun` method is invoked with an `InteractionContext`. This context provides access to the participating entities and a `CommandBuffer`, a standard ECS pattern for queueing state changes to ensure deterministic, thread-safe updates to the game world.

A key feature is its extensible validation mechanism. The static `USAGE_VALIDATOR_MAP` acts as a central registry for custom logic. This allows other engine modules to register `PageUsageValidator` functions, injecting conditional checks (e.g., does the player have the required quest status?) before the page is opened. This is a form of the **Strategy Pattern** that promotes high cohesion and low coupling between the interaction system and other game systems.

## Lifecycle & Ownership

-   **Creation:** Instances are not created directly using the `new` keyword in game logic. They are instantiated by the `BuilderCodec` system during server startup or when game assets are loaded. The engine parses configuration files that define interactions and uses the static `CODEC` field to construct and populate `OpenPageInteraction` objects.
-   **Scope:** An instance is effectively a stateless template that persists for the lifetime of the server or until game assets are reloaded. It is shared and reused for every interaction of its type.
-   **Destruction:** Instances are garbage collected when their containing configuration is unloaded, typically during an asset reload or server shutdown.

## Internal State & Concurrency

-   **State:** An `OpenPageInteraction` instance is **effectively immutable** after its creation by the codec. Its fields, `page` and `canCloseThroughInteraction`, are configured once during deserialization and are not modified during gameplay. The class itself is stateless; all contextual information required for execution is provided via the `InteractionContext` argument in the `firstRun` method.
-   **Thread Safety:** The class is inherently thread-safe.
    -   Instance-level safety is guaranteed by its immutable nature.
    -   The static `USAGE_VALIDATOR_MAP` uses a `ConcurrentHashMap`, making the registration of validators safe to perform from multiple threads during server initialization.
    -   The `firstRun` method achieves thread safety by delegating all world-state mutations to the `CommandBuffer`, which serializes operations to prevent race conditions within the ECS.

## API Surface

The primary public contract is the static registration method, as `firstRun` is part of an internal contract with the Interaction System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | **Internal Contract.** Executes the interaction. Retrieves the Player component and, after optional validation, commands the player's PageManager to open the configured UI page. |
| registerUsageValidator(page, validator) | static void | O(1) | Registers a custom validation function for a specific Page. This is the primary extension point for this class. |

## Integration Patterns

### Standard Usage

This class is intended to be used declaratively in configuration files. A developer's primary interaction with it is to register custom validation logic during a module's initialization phase.

```java
// In a module's initialization block
// This validator prevents the Crafting Table page from opening if a custom condition is not met.

OpenPageInteraction.registerUsageValidator(
    Page.CRAFTING_TABLE,
    (entityRef, player, context, accessor) -> {
        // Example: Check if the player has a specific "Tinkerer" component
        return accessor.hasComponent(entityRef, Tinkerer.getComponentType());
    }
);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new OpenPageInteraction()`. Instances must be created via the `Codec` system from data files. Direct instantiation will result in a non-functional object that is not registered with the game's interaction systems.
-   **Direct Invocation:** Do not call the `firstRun` method directly. It must be invoked by the server's core interaction processor, which is responsible for supplying a valid `InteractionContext` and `CommandBuffer`. Bypassing the interaction system will break ECS conventions and likely cause world state corruption.
-   **Runtime Validator Modification:** Avoid calling `registerUsageValidator` after the server has completed its initial startup. While technically thread-safe, modifying validators during active gameplay can lead to unpredictable and non-deterministic behavior for players.

## Data Pipeline

The flow of data and control begins with a game asset definition and ends with a UI update on the player's client.

> Flow:
> 1. **Game Asset (JSON)**: A designer defines an interaction on a block, specifying `OpenPageInteraction` and its properties.
> 2. **Server Startup**: The `Codec` system deserializes the JSON into an in-memory `OpenPageInteraction` object.
> 3. **Player Action**: A player in-game interacts with the configured block or entity.
> 4. **Interaction System**: The core system receives the event and invokes the `firstRun` method on the corresponding `OpenPageInteraction` instance.
> 5. **Validation**: The `firstRun` method checks the static `USAGE_VALIDATOR_MAP` for a registered validator for the target page. If one exists, it is executed.
> 6. **Command Buffer**: If validation succeeds, a command to open the page is recorded in the `CommandBuffer`.
> 7. **ECS Tick**: The `CommandBuffer` is flushed at the end of the server tick, safely applying the change to the `Player` component's `PageManager`.
> 8. **Network Packet**: The `PageManager` serializes a packet to the client, instructing it to open the new UI page.


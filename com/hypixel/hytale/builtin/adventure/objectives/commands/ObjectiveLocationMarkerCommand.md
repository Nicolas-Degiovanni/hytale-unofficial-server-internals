---
description: Architectural reference for ObjectiveLocationMarkerCommand
---

# ObjectiveLocationMarkerCommand

**Package:** com.hypixel.hytale.builtin.adventure.objectives.commands
**Type:** Utility

## Definition
```java
// Signature
public class ObjectiveLocationMarkerCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ObjectiveLocationMarkerCommand class serves as a centralized command dispatcher for all server-side operations related to objective location markers. It follows the Command pattern, encapsulating discrete actions into a structured, discoverable hierarchy for server administrators and adventure mode scripters.

Architecturally, this class is a "collection command". Its sole purpose is to group related sub-commands—`add`, `enable`, and `disable`—under a unified namespace: `locationmarker`. This prevents pollution of the global command space and provides a clear, organized interface for a specific feature set.

The sub-commands operate on two distinct layers of the server architecture:
1.  **Entity Layer:** The `AddLocationMarkerCommand` directly manipulates the game world's state by interacting with the Entity Component System (ECS). It constructs a new entity from a series of components (e.g., TransformComponent, ModelComponent, ObjectiveLocationMarker) and injects it into the world's `EntityStore`.
2.  **Configuration Layer:** The `EnableLocationMarkerCommand` and `DisableLocationMarkerCommand` modify persistent world settings. They interact with the `WorldConfig` object to toggle a global flag that controls the rendering or behavior of all objective markers within that world.

This separation of concerns within the sub-commands allows for both fine-grained entity creation and broad, world-level configuration changes through a single, cohesive command structure.

### Lifecycle & Ownership
-   **Creation:** An instance of ObjectiveLocationMarkerCommand is created once during the server's bootstrap sequence. It is typically instantiated and registered by its parent feature plugin, `ObjectivePlugin`, with the server's central `CommandSystem`.
-   **Scope:** The object is stateless and long-lived. It persists for the entire server session, held as a reference within the command registry. The logic within its sub-commands is executed on-demand when a corresponding command is invoked.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class and its nested sub-commands are fundamentally stateless. All required information for execution is provided at runtime through the `CommandContext`, `World`, and `Store` parameters. The static `Message` fields are immutable, compile-time constants used for localization.

-   **Thread Safety:** **Not thread-safe.** Command execution is exclusively managed by the server's main thread. All interactions with the ECS `Store` and `World` objects are inherently unsafe if performed from any other thread. Any attempt to invoke command logic asynchronously will lead to severe concurrency issues, including `ConcurrentModificationException` and world state corruption. This is a strict engine-level constraint.

## API Surface
The primary programmatic interface is the constructor, used for registration. The user-facing API consists of the console commands it makes available.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ObjectiveLocationMarkerCommand() | constructor | O(1) | Initializes the command collection, setting its primary name and registering all sub-commands. |

## Integration Patterns

### Standard Usage
This class is not designed for direct programmatic invocation. Its functionality is exposed exclusively through the server's command-line interface. Developers and server administrators should interact with it via in-game chat or the server console.

```
# Creates a new objective marker entity at the player's current position.
/objective locationmarker add my_zone1_marker

# Enables the rendering of all objective markers in the current world.
/objective locationmarker enable

# Disables the rendering of all objective markers in the current world.
/objective locationmarker disable
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ObjectiveLocationMarkerCommand()` in game logic. The object provides no useful methods and will have no effect unless it is properly registered with the server's command system during startup.
-   **Stateful Implementation:** Do not modify this class to hold mutable instance state. Commands must remain stateless to ensure they are re-entrant and produce predictable results regardless of execution history.
-   **Asynchronous Execution:** Never attempt to execute command logic from a separate thread. All world and entity modifications must be synchronized with the main server tick.

## Data Pipeline
The data flow for this class is initiated by user input and results in a direct mutation of server state. The specific pipeline depends on the sub-command executed.

**AddLocationMarkerCommand Pipeline:**
> Flow:
> Player Command Input -> Server Command Parser -> **AddLocationMarkerCommand::execute** -> Entity Holder Creation -> Component Assembly (`Transform`, `Model`, `ObjectiveLocationMarker`) -> `EntityStore::addEntity` -> World State Mutation -> Player Feedback Message

**Enable/DisableLocationMarkerCommand Pipeline:**
> Flow:
> Player Command Input -> Server Command Parser -> **Enable/DisableLocationMarkerCommand::execute** -> `World::getWorldConfig` -> `WorldConfig::setObjectiveMarkersEnabled` -> `WorldConfig::markChanged` (Serialization) -> Persistent World State Mutation -> Player Feedback Message


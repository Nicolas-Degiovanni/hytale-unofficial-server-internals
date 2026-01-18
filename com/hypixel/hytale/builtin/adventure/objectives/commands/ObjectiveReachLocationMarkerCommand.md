---
description: Architectural reference for ObjectiveReachLocationMarkerCommand
---

# ObjectiveReachLocationMarkerCommand

**Package:** com.hypixel.hytale.builtin.adventure.objectives.commands
**Type:** Utility

## Definition
```java
// Signature
public class ObjectiveReachLocationMarkerCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The ObjectiveReachLocationMarkerCommand class serves as a command-line interface bridge to the server's Entity Component System (ECS). It does not contain game logic itself; rather, it acts as a container or namespace for subcommands related to managing `ReachLocationMarker` entities. Its primary architectural function is to translate administrative text commands into concrete entity creation and manipulation within the game world.

This class follows the Command pattern, specifically as a collection that groups related actions. The core logic is implemented in the nested `AddReachLocationMarkerCommand` class, which handles the parsing of arguments and the subsequent assembly of a new entity. This entity is purpose-built as a physical marker for the adventure mode objective system, composed of several distinct components that define its appearance, behavior, and persistence.

## Lifecycle & Ownership

-   **Creation:** An instance of ObjectiveReachLocationMarkerCommand is created by the server's command registration system during the server bootstrap phase or when the associated plugin is loaded. The system discovers all classes extending `AbstractCommandCollection` and registers them for handling.
-   **Scope:** The registered command object is a stateless singleton managed by the command registry. It persists for the entire server session. The execution context of the command, however, is transient and scoped to a single invocation.
-   **Destruction:** The object is de-registered and becomes eligible for garbage collection only when the server shuts down or the parent plugin is unloaded.

## Internal State & Concurrency

-   **State:** This class is stateless and effectively immutable. Its fields are `final` references to pre-configured `Message` objects used for sending feedback to the command issuer. All state required for execution, such as the target world and the player's position, is provided at runtime through the `CommandContext` and other method parameters.
-   **Thread Safety:** The class itself is thread-safe. However, the `execute` method of its subcommand performs mutations on the world's `EntityStore`. The Hytale command system guarantees that all command executions are serialized and processed on the main server thread. This prevents race conditions and concurrent modification exceptions within the ECS.

    **Warning:** Manually invoking the `execute` method from an asynchronous task or a non-server thread will lead to world state corruption and is strictly forbidden.

## API Surface

The public API is designed for consumption by the internal command system, not for general developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AddReachLocationMarkerCommand.execute(...) | void | O(1) | Executes the command logic. Validates the marker ID, constructs a new entity with multiple components, and adds it to the world at the player's current location. |

## Integration Patterns

### Standard Usage

This class is not intended to be instantiated or called directly from game logic code. It is invoked exclusively by the server's command parser in response to a player or administrator typing the command into the console.

The standard interaction is via the in-game command line:
`/objective reachlocationmarker add <marker_asset_id>`

Example:
```plaintext
/objective reachlocationmarker add kraytons_folly_entrance
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ObjectiveReachLocationMarkerCommand()`. The object is useless unless registered with the server's command system, which is handled automatically.
-   **Manual Execution:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including argument parsing, permission validation, and the thread synchronization provided by the command dispatcher. This is a direct path to server instability.

## Data Pipeline

The flow of data for this command begins with user input and ends with a world state change and user feedback.

> Flow:
> Player Command String -> Server Network Parser -> **Command Dispatcher** -> Argument Validation (vs ReachLocationMarkerAsset) -> **AddReachLocationMarkerCommand.execute** -> Entity Holder Creation -> Component Assembly (Transform, Model, etc.) -> EntityStore.addEntity -> World State Update -> Confirmation Message -> Player
---


---
description: Architectural reference for PrefabEditSelectCommand
---

# PrefabEditSelectCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditSelectCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabEditSelectCommand is a server-side command handler responsible for processing player input to select a prefab within an active editing session. It is a component of the **Builder Tools Plugin**, a suite of features designed for in-game content creation and world editing.

This class acts as the primary user-facing entry point for modifying the selection state within a PrefabEditSession. It translates a player's action—either looking at a prefab or using a proximity flag—into a state change on their corresponding session object. The command queries the game world to identify the target prefab entity and then delegates the actual state mutation to the PrefabEditSessionManager, ensuring a clean separation of concerns between user input handling and session state management.

Two distinct selection strategies are supported:
1.  **Target-Based Selection:** The default mode. It performs a raycast from the player's camera to find the block or entity they are looking at. It then checks if this location is within the bounding box of any loaded prefabs.
2.  **Proximity-Based Selection:** Activated by the *--nearest* flag. It calculates the 2D distance from the player to the center of each loaded prefab's bounding box and selects the closest one.

## Lifecycle & Ownership
-   **Creation:** An instance of PrefabEditSelectCommand is created and registered by the server's command system during the initialization of the BuilderToolsPlugin. It is not instantiated per-execution.
-   **Scope:** The command object is a long-lived singleton that persists for the entire server session, or until the parent plugin is unloaded.
-   **Destruction:** The object is garbage collected when the BuilderToolsPlugin is disabled and the command registry releases its reference, typically during server shutdown.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its fields, such as nearestArg, are initialized in the constructor and are not mutated during command execution. All stateful operations are performed on objects passed into the execute method (like CommandContext and Store) or on the player's PrefabEditSession, which is managed externally.
-   **Thread Safety:** **Not Thread-Safe.** This command is designed to be executed exclusively by the server's main game thread or a designated command processing thread. It accesses and modifies game state (Player, World, PrefabEditSession) that is not thread-safe. The command system's execution model guarantees that it is not invoked concurrently for the same player, preventing race conditions.

## API Surface
The public contract is defined by its inheritance from AbstractPlayerCommand. The primary entry point is the protected execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the selection logic. Complexity is O(N) where N is the number of prefabs loaded in the player's session, due to the iteration required to find the target. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically invoked by the server's command system when a player executes the command in-game.

The intended interaction is through the game client's chat or console:
```
/prefab edit select
```
Or with the proximity flag:
```
/prefab edit select --nearest
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabEditSelectCommand()`. The command must be registered with the server's command system to function. Creating an instance manually has no effect.
-   **Direct Invocation:** Do not call the `execute` method directly. This bypasses critical infrastructure provided by the command system, including context setup, permission checks, and argument parsing.
-   **External Session Modification:** Do not modify a player's PrefabEditSession from another thread while this command might be executing. The session object is not designed for concurrent access.

## Data Pipeline
The command processes data by acting as a bridge between player input, the game world, and the prefab editing session state.

> Flow:
> Player Command Input (`/prefab edit select`) -> Server Command Dispatcher -> **PrefabEditSelectCommand.execute()** -> World Query (via TargetUtil or proximity calculation) -> PrefabEditSessionManager -> PrefabEditSession.setSelectedPrefab() -> Player Feedback Message


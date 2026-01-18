---
description: Architectural reference for SpawnMarkersCommand
---

# SpawnMarkersCommand

**Package:** com.hypixel.hytale.server.spawning.commands
**Type:** Utility

## Definition
```java
// Signature
public class SpawnMarkersCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The SpawnMarkersCommand class serves as a command aggregator, providing a unified command-line interface for managing spawn markers within a game world. It does not implement any game logic itself; instead, it acts as a namespace and registration hub for a collection of specialized sub-commands. This design adheres to the Command pattern, where distinct actions are encapsulated into their own objects.

This class bridges the server's high-level command processing system with the low-level world state and Entity Component System (ECS). When a user executes a command like `/markers add`, the server's command dispatcher routes the request to the appropriate sub-command registered within this collection.

The primary sub-commands are:
*   **Add:** An entity factory command. It is responsible for constructing and injecting a new SpawnMarkerEntity into the world's EntityStore. It assembles the entity from a series of components, including TransformComponent, ModelComponent, and Nameplate, based on arguments provided by the user.
*   **EnableCommand / DisableCommand:** World state mutator commands. These directly modify a boolean flag within the active WorldConfig object, enabling or disabling the spawn marker system globally for that world. This change is persisted with the world state.

## Lifecycle & Ownership

*   **Creation:** A single instance of SpawnMarkersCommand is created by the server's core CommandSystem during the server bootstrap phase. The system discovers and instantiates all classes that extend command-related base classes.
*   **Scope:** The instance persists for the entire server session. It is held in a central command registry and is not subject to garbage collection until server shutdown. The sub-command instances (Add, EnableCommand, DisableCommand) are created in the constructor and share the same lifecycle as the parent collection.
*   **Destruction:** The object is dereferenced and cleaned up only when the server shuts down and the CommandSystem is dismantled.

## Internal State & Concurrency

*   **State:** This class is effectively stateless and immutable after construction. Its only internal state is the list of sub-commands it manages, which is populated once in the constructor and never modified again. The static AssetArgumentType field is a shared, immutable definition for parsing command arguments.
*   **Thread Safety:** Command execution is managed by the server's main thread. The `execute` methods of the inner command classes are not designed for concurrent access and must only be invoked by the engine's command dispatcher. All interactions with the World and EntityStore are performed within the server's single-threaded game loop, which guarantees atomicity at the tick level and obviates the need for explicit locking.

## API Surface

The primary programmatic interaction is instantiation by the command framework. The user-facing API consists of the chat commands it registers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SpawnMarkersCommand() | Constructor | O(1) | Instantiates the command collection and registers its sub-commands. Intended for framework use only. |

## Integration Patterns

### Standard Usage

This class is not intended to be used directly in code. Interaction occurs through the server console or in-game chat by an authorized user.

**Example: Creating a new spawn marker**
```
/markers add my_dungeon:goblin_spawner --flip
```
This command invokes the `Add` sub-command, creating a new spawn marker entity using the specified asset and applying a 180-degree rotation on the Y-axis.

**Example: Enabling the system for the current world**
```
/markers enable
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not create an instance via `new SpawnMarkersCommand()` in game logic. The object is inert unless registered with the server's CommandSystem. The framework handles its lifecycle exclusively.
*   **Direct Execution:** Never call the `execute` method on the inner command classes directly. These methods require a fully-formed CommandContext, World, and EntityStore reference, which are only supplied by the command processing pipeline. Bypassing the pipeline can lead to inconsistent world state or server instability.

## Data Pipeline

The data flow for the most complex operation, the `Add` command, illustrates how user input is transformed into a world state change.

> Flow:
> Player Chat Input (`/markers add ...`) -> Network Layer -> Server Command Parser -> **SpawnMarkersCommand** (Routing) -> **Add.execute()** -> EntityStore.REGISTRY.newHolder() -> Component Assembly (Transform, Model, etc.) -> EntityStore.addEntity() -> World State Commit


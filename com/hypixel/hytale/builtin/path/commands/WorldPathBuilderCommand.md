---
description: Architectural reference for WorldPathBuilderCommand
---

# WorldPathBuilderCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Command Collection

## Definition
```java
// Signature
public class WorldPathBuilderCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The WorldPathBuilderCommand class serves as the primary user-facing entry point for creating and editing WorldPath objects on a game server. It is not a persistent service or manager; rather, it is a stateless command dispatcher that registers a collection of related sub-commands under the primary `/worldpath builder` namespace.

The core architectural pattern is a separation of command logic from session state. This class and its nested sub-commands contain only the execution logic. The state of an active path-building session is managed using the Entity Component System (ECS). When a player initiates a build session (e.g., by using the `add` sub-command), a temporary **WorldPathBuilder** component is attached to that player's entity in the world's EntityStore.

Subsequent commands from that player operate on this component. For example, `add` appends the player's current transform to the waypoint list within the component, and `remove` deletes an entry from that same list. This design elegantly ties the build session's lifetime to the player's presence in the world and ensures that the command handlers themselves remain stateless and reusable.

Finalization of the path occurs when the `save` command is executed. This command retrieves the data from the WorldPathBuilder component, constructs a permanent WorldPath object, removes the temporary component from the player entity, and hands the WorldPath over to the WorldPathConfig for serialization and persistence.

## Lifecycle & Ownership

-   **Creation:** A single instance of WorldPathBuilderCommand is instantiated by the server's command registration system during the server bootstrap phase. It is discovered and registered automatically as part of the server's core command set.
-   **Scope:** The WorldPathBuilderCommand object is a stateless singleton that persists for the entire lifetime of the server process.
-   **Destruction:** The object is garbage collected during server shutdown.

**WARNING:** The lifecycle described above applies only to the command collection object itself. The state it manages—the **WorldPathBuilder** component—has a separate, more transient lifecycle. A WorldPathBuilder component is created on a player entity on-demand and is destroyed when the player logs off, or when the `save`, `stop`, or `clear` commands are executed.

## Internal State & Concurrency

-   **State:** The WorldPathBuilderCommand class and all its nested sub-command classes are **stateless**. They do not hold any mutable fields related to a specific player's build session. All session state is externalized and stored within a WorldPathBuilder component, which is attached to a player entity in the ECS EntityStore. This component is highly mutable during an active build session.

-   **Thread Safety:** This class is **not thread-safe** for direct invocation. The Hytale server architecture guarantees that all command execution occurs on the main world thread. The static helper methods within this class (e.g., getBuilder, putBuilder) directly access the EntityStore and are therefore unsafe to call from any other thread.

    The `WorldPathBuilderSimulateCommand` is a notable exception. It uses the global HytaleServer.SCHEDULED_EXECUTOR to schedule tasks on a background thread pool. However, it correctly maintains thread safety by dispatching all entity modifications back to the main world thread using a `world.execute` lambda. This is the mandatory pattern for performing entity operations from an asynchronous context.

## API Surface

The public contract of this class is exposed as a set of chat commands available to server operators, not as a programmatic Java API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add | Sub-Command | O(1) | Adds the player's current transform as a new waypoint to the active build session. |
| clear | Sub-Command | O(N) | Clears all waypoints from the active build session. N is the number of waypoints. |
| goto | Sub-Command | O(1) | Teleports the player to the transform of a specified waypoint index. |
| load | Sub-Command | O(M) | Loads an existing WorldPath into a new build session, overwriting any current session. M is the number of waypoints in the loaded path. |
| remove | Sub-Command | O(N) | Removes the waypoint at the specified index. N is the number of waypoints. |
| save | Sub-Command | O(N) | Saves the current waypoints as a new, named WorldPath, ending the build session. |
| set | Sub-Command | O(1) | Overwrites the waypoint at a specified index with the player's current transform. |
| simulate | Sub-Command | O(1) | Initiates an asynchronous task that teleports the player along the path's waypoints. |
| stop | Sub-Command | O(1) | Discards the current build session without saving. |

## Integration Patterns

### Standard Usage

This class is designed for interaction via the in-game command line by a server administrator. There is no public API intended for direct developer consumption.

```sh
# Example command-line flow for a server administrator

# Start a new path and add the first point
/worldpath builder add

# Move to a new location and add another point
/worldpath builder add

# Teleport back to the first point (index 0) to check it
/worldpath builder goto 0

# Save the path with the name "PatrolRoute1"
/worldpath builder save PatrolRoute1
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not create an instance with `new WorldPathBuilderCommand()`. The object is useless unless registered with the server's command system, which is handled internally.
-   **Asynchronous State Modification:** Do not call the static helper methods (`getBuilder`, `putBuilder`, `removeBuilder`) from a background thread. These methods manipulate the EntityStore and must be called from the main world thread, typically within a `world.execute()` block. Failure to do so will result in concurrency violations and data corruption.
-   **State Assumption:** Do not assume a WorldPathBuilder component exists on a player entity. Always check for null, as a player may not have an active build session. The `getOrCreateBuilder` method is the safe entry point for commands that can initiate a session.

## Data Pipeline

The primary data flow is initiated by user input and results in either a modification to an ECS component or a write operation to the filesystem.

> **Flow (Adding a waypoint):**
> User Input (`/wp builder add`) → Server Command Parser → `WorldPathBuilderCommand` Dispatcher → `WorldPathBuilderAddCommand.execute` → Read Player TransformComponent → Modify or Create **WorldPathBuilder** Component → Commit to EntityStore → Send Feedback Message to User

> **Flow (Saving a path):**
> User Input (`/wp builder save MyPath`) → Server Command Parser → `WorldPathBuilderCommand` Dispatcher → `WorldPathBuilderSaveCommand.execute` → Read **WorldPathBuilder** Component → Remove Component from EntityStore → `World.getWorldPathConfig().putPath()` → `WorldPathConfig.save()` → Filesystem Write Operation → Send Feedback Message to User


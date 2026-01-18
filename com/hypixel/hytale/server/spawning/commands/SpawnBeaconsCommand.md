---
description: Architectural reference for SpawnBeaconsCommand
---

# SpawnBeaconsCommand

**Package:** com.hypixel.hytale.server.spawning.commands
**Type:** Utility

## Definition
```java
// Signature
public class SpawnBeaconsCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The SpawnBeaconsCommand class serves as a command-line interface for server administrators to interact with the NPC spawn beacon system. It is not a standalone command but rather a composite command collection, grouping related functionalities under a single `beacons` namespace. This design pattern simplifies command registration and user interaction by organizing sub-commands like `add` and `trigger` into a logical hierarchy.

Architecturally, this class acts as a bridge between the server's Command System and the core logic of the SpawningPlugin and the Entity-Component-System (ECS). It does not implement any spawning logic itself. Instead, it parses arguments, retrieves context from the command executor (such as the player's location), and then directly manipulates the world's `EntityStore` to create or trigger spawn beacon entities.

The primary responsibilities of this class are:
1.  **Command Registration:** It registers the primary `beacons` command and its sub-commands with the server's central command handler.
2.  **Entity Factory:** The `Add` sub-command functions as a factory for creating `SpawnBeacon` or `LegacySpawnBeaconEntity` entities. It manually assembles the required components (e.g., TransformComponent, ModelComponent, DisplayNameComponent) and adds the new entity to the `EntityStore`.
3.  **System Invocation:** The `ManualTrigger` sub-command retrieves existing beacon entities from the `EntityStore` and invokes their spawning behavior, effectively delegating the complex task of finding spawn locations and creating NPCs to the appropriate systems.

### Lifecycle & Ownership
-   **Creation:** A single instance of SpawnBeaconsCommand is created by the server's command registration system during the server bootstrap sequence. The system scans for classes extending `AbstractCommandCollection` or similar base types and instantiates them.
-   **Scope:** The instance persists for the entire server session. It is held in a central command registry and is never destroyed until the server shuts down. The execution of its sub-commands, however, is transient and scoped to a single command invocation.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only during server shutdown when the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is designed to be stateless. All necessary information for execution (e.g., the target world, the executing player, command arguments) is provided via the `CommandContext` and other parameters passed into the `execute` method of its sub-commands. The only persistent fields are final references to argument types and sub-command instances, which are immutable after construction.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed by the main server thread. The Hytale server architecture guarantees that all command executions occur serially on the primary game loop thread. Any attempt to invoke the `execute` method from an asynchronous task or worker thread will lead to race conditions and severe world state corruption. All interactions with the `EntityStore` are fundamentally unsafe outside the main tick loop.

## API Surface

The public contract of this class is exposed via the server command line, not through direct method calls. The following sub-commands constitute its primary interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add | Command | O(1) | Creates a new spawn beacon entity at the player's location. Assembles multiple components and adds a new entity to the world's EntityStore. |
| trigger | Command | O(N) | Manually triggers the spawning logic of the targeted spawn beacon. Complexity is dependent on the FloodFillPositionSelector's area scan. Throws GeneralCommandException if the target is not a valid beacon. |

## Integration Patterns

### Standard Usage

This class is intended for use by server operators via the in-game console or chat interface. It is not designed to be invoked programmatically from other plugins or systems.

**Example: Creating a new manual-trigger beacon**
```
/beacons add my_goblin_spawn --manual
```

**Example: Forcing a targeted beacon to spawn NPCs**
```
/beacons trigger
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new SpawnBeaconsCommand()`. The server's command system is solely responsible for the lifecycle of command objects. Direct instantiation will result in a non-functional command that is not registered with the server.
-   **Programmatic Invocation:** Do not attempt to get a reference to this class to call its `execute` methods directly. This bypasses critical infrastructure, including permission checks, context setup, and argument parsing, and is an unsupported and dangerous practice. Use the dedicated `CommandSystem` API to dispatch commands if programmatic execution is required.
-   **Stateful Modification:** Do not modify this class to hold state between executions. Commands must be stateless to ensure predictable behavior in a server environment.

## Data Pipeline

The flow of data for this class begins with user input and ends with a modification to the world state. It acts as a controller that translates text commands into direct ECS operations.

**`add` Sub-Command Flow:**
> Player Chat Input (`/beacons add...`) -> Network Layer -> Command Parser -> **SpawnBeaconsCommand.Add.execute()** -> EntityStore.newHolder() -> Component Assembly (Transform, Model, etc.) -> EntityStore.addEntity() -> World State Change

**`trigger` Sub-Command Flow:**
> Player Chat Input (`/beacons trigger`) -> Network Layer -> Command Parser -> **SpawnBeaconsCommand.ManualTrigger.execute()** -> EntityStore.getComponent(targetEntity) -> SpawnBeacon.manualTrigger() -> Spawning System Logic -> EntityStore.addEntity() (for new NPCs) -> World State Change


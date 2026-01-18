---
description: Architectural reference for WorldMapDiscoverCommand
---

# WorldMapDiscoverCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Singleton Command Handler

## Definition
```java
// Signature
public class WorldMapDiscoverCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WorldMapDiscoverCommand class is a server-side implementation of the Command pattern, designed to be integrated into the server's central command processing system. Its primary architectural role is to act as a translator between player chat input and direct state manipulation within the server's Entity-Component-System (ECS).

This command specifically targets the world map discovery state of a Player entity. It retrieves world data from the WorldMapManager, compares it against the target player's persisted discovery data held in the Player component, and performs mutations on that component via its WorldMapTracker.

It is a leaf node in the server's logic, responsible for a single, discrete administrative action. It does not hold or manage long-term state; rather, it is a stateless service that operates on state passed to it by the command system and the ECS.

## Lifecycle & Ownership
- **Creation:** A single instance of WorldMapDiscoverCommand is instantiated by the server's command registration service during the server bootstrap sequence. It is registered under the name "discover" as a subcommand of "worldmap".

- **Scope:** The object instance is a singleton that persists for the entire lifetime of the server. However, its *execution context* is transient and scoped to a single command invocation. The class is designed to be stateless between calls to its execute method.

- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The class instance is effectively immutable after construction. It contains final fields for its argument definitions (zoneArg) and pre-compiled message templates. All state required for execution, such as the player entity and the world state, is provided as parameters to the execute method. It performs no internal caching.

- **Thread Safety:** **This class is not thread-safe.** The execute method performs direct, unsynchronized read and write operations on ECS components (specifically the Player component and its associated WorldMapTracker). The Hytale server architecture mandates that all ECS interactions occur on the main server thread.

   **WARNING:** Invoking the execute method from any thread other than the main server tick thread will result in critical race conditions, data corruption, and unpredictable server crashes.

## API Surface
The public contract is defined by its role as a command handler. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | protected void | O(N) | Executes the command logic. N is the number of defined biome zones in the world. Complexity is O(1) for a single zone lookup but O(N) for the "all" case or for fuzzy string matching on failure. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked exclusively by the server's command system in response to player input. The standard interaction is via the in-game chat console.

```sh
# List all available zones and undiscovered zones for the player
/worldmap discover

# Discover a specific zone by name
/worldmap discover "Emerald Grove"

# Discover all available zones at once
/worldmap discover all
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldMapDiscoverCommand()`. The command system manages the lifecycle of this object. Bypassing the system will lead to a non-functional command that is not registered to handle any input.

- **Manual Execution:** Do not manually invoke the `execute` method. This bypasses critical infrastructure provided by the CommandContext, including permission checks, argument parsing, and player feedback mechanisms.

- **Asynchronous Access:** As detailed under Concurrency, never call this class from a separate thread. All interactions must be scheduled on the main server thread.

## Data Pipeline
The flow of data for a successful command execution follows a clear path from player input to state change and back to player feedback.

> Flow:
> Player Chat Input (`/worldmap discover all`) -> Network Packet -> Server Command Parser -> **WorldMapDiscoverCommand.execute()** -> World.getWorldMapManager() -> ECS Store -> Player Component & WorldMapTracker (State Mutation) -> CommandContext.sendMessage() -> Network Packet -> Client Chat UI Update


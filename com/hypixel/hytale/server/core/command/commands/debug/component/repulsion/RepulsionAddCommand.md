---
description: Architectural reference for RepulsionAddCommand
---

# RepulsionAddCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.component.repulsion
**Type:** Transient

## Definition
```java
// Signature
public class RepulsionAddCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The RepulsionAddCommand class is a command definition that exposes administrative functionality for the server's Entity Component System (ECS). It is not a long-running service but rather a declarative structure that registers a command hierarchy with the server's central command processor.

Architecturally, this class implements the **Composite Pattern**. The top-level RepulsionAddCommand acts as a container or a router for its nested sub-commands: RepulsionAddEntityCommand and RepulsionAddSelfCommand. This design creates the user-facing command structure `repulsion add <subcommand>`, allowing for modular and extensible command definitions.

The primary role of this command is to act as a bridge between user input (a player or console command) and direct state mutation within the game world. It translates a text-based command into a specific ECS operation: adding a Repulsion component to a target entity. This allows developers and server administrators to dynamically alter entity behavior for debugging and testing purposes without modifying game code.

### Lifecycle & Ownership
- **Creation:** An instance of RepulsionAddCommand is created once during server initialization by the command registration system. The system discovers all command classes and instantiates them to build a registry of available commands.
- **Scope:** The command definition object is application-scoped. It persists in memory for the entire duration of the server session, held as a reference within the central CommandRegistry.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively **immutable and stateless** after its constructor is run. Its fields consist of final Message constants and instances of its sub-commands, which are also stateless definitions. The command does not hold any data related to a specific execution; all necessary state is passed into the `execute` methods via the CommandContext and World parameters.
- **Thread Safety:** The command definition object is thread-safe and can be safely stored in a shared, concurrently accessed registry. However, the `execute` methods of its sub-commands are designed to be invoked by the server's main game loop or a designated command-processing thread. These methods perform direct mutations on the World's EntityStore and are **not safe** to be called concurrently for the same World instance. The engine's architecture guarantees that command execution is synchronized with the world tick, preventing race conditions at the ECS level.

## API Surface
The primary API is the contract inherited from the command system, specifically the `execute` methods within the nested command classes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RepulsionAddEntityCommand.execute | void | O(1) | Adds a Repulsion component to a specified entity. Throws exceptions if arguments are malformed. Sends feedback messages to the command source. |
| RepulsionAddSelfCommand.execute | void | O(1) | Adds a Repulsion component to the entity associated with the command's source. Sends feedback messages to the command source. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly via Java code. Its functionality is exposed exclusively through the server's command-line interface (in-game chat or server console).

**To add a repulsion component to a specific entity:**
```
/repulsion add entity <entityId> <repulsionConfig>
```

**To add a repulsion component to yourself:**
```
/repulsion add self <repulsionConfig>
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance using `new RepulsionAddCommand()` in game logic. Command objects are managed by the command system and have no effect if instantiated outside of the registration process.
- **Manual Execution:** Never call the `execute` method directly. It requires a fully populated CommandContext and a valid World state, which are only provided by the command processing system. Bypassing the system will lead to NullPointerExceptions and world state corruption.

## Data Pipeline
This command initiates a one-way data flow that results in a change to the world state. The physics engine then consumes this new state on subsequent ticks.

> Flow:
> User Input (`/repulsion add...`) -> Command Parser -> **RepulsionAddCommand** (Routing) -> Sub-command `execute` method -> `Store.addComponent(entity, Repulsion)` -> Entity Component System State Change -> Physics System (reads new Repulsion component) -> Modified Entity Behavior


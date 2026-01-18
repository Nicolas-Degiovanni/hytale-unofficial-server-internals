---
description: Architectural reference for WorldTpsCommand
---

# WorldTpsCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world.tps
**Type:** Transient

## Definition
```java
// Signature
public class WorldTpsCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldTpsCommand class is an implementation of the Command Pattern, encapsulating the logic required to modify a game world's Ticks Per Second (TPS) rate. It serves as a direct interface between a user-invoked action (like a console command) and the core state of a running world simulation.

By extending AbstractWorldCommand, this class integrates into a framework that automatically provides it with the necessary context for execution, namely the specific **World** instance it is meant to operate on. This design decouples the command's logic from the complex machinery of command parsing, argument resolution, and world lookups. Its primary architectural role is to act as a transactional controller: it receives a request via the CommandContext, validates the input parameters, applies a state change to the World model, and provides feedback to the user.

This class also demonstrates a compositional structure by registering a sub-command, WorldTpsResetCommand, creating a hierarchical command tree (e.g., `/world tps` and `/world tps reset`).

## Lifecycle & Ownership
- **Creation:** An instance of WorldTpsCommand is created once during server initialization or plugin loading. It is then registered with a central CommandRegistry, which retains ownership of the instance.
- **Scope:** The command object itself is a long-lived singleton for the duration of the server's runtime. However, its execution context is ephemeral; the `execute` method is invoked for a brief, finite period each time the command is triggered by a user.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server's CommandRegistry is cleared, typically during a full server shutdown.

## Internal State & Concurrency
- **State:** This class is effectively stateless and immutable after its constructor has run. Its fields are final references to message templates and an argument definition object. It does not cache data or maintain any state between executions. All state it operates on is external, passed into the `execute` method via the World and CommandContext parameters.

- **Thread Safety:** **This class is not thread-safe.** The `execute` method is designed to be invoked exclusively by the server's main thread, which has synchronous access to the game loop and world state. Modifying the World object from any other thread would bypass critical engine safeguards and lead to severe data corruption, race conditions, and simulation instability. All interactions with the World object must be synchronized with the main server tick loop.

## API Surface
The public contract is primarily defined by its constructor for registration and the `execute` method for invocation by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldTpsCommand() | constructor | O(1) | Creates the command, defining its name, alias, and sub-commands. |
| execute(context, world, store) | void | O(1) | Executes the TPS change. Mutates the passed World object. Throws exceptions if context is invalid. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic invocation. It is designed to be registered with the server's command system, which handles its lifecycle and execution.

```java
// Example of registering the command during server startup
CommandRegistry commandRegistry = server.getCommandSystem().getRegistry();
commandRegistry.register(new WorldTpsCommand());

// A user would then execute this via the console or chat:
// > /world tps 60
```

### Anti-Patterns (Do NOT do this)
- **Direct Execution:** Never call the `execute` method directly. This bypasses the command system's argument parsing, permission checks, and context resolution, and it can lead to unpredictable behavior.
  ```java
  // ANTI-PATTERN: Bypasses the entire command framework
  WorldTpsCommand cmd = new WorldTpsCommand();
  cmd.execute(myContext, myWorld, myStore); // DANGEROUS
  ```
- **Asynchronous Invocation:** Do not invoke this command from a separate thread. The `World` object is not thread-safe and must only be modified from the main server thread.

## Data Pipeline
The flow of data for this command is initiated by an external user and results in a direct mutation of a core server object.

> Flow:
> User Input (`/world tps 60`) -> Network Layer -> Command Parser -> **WorldTpsCommand.execute()** -> World.setTps(60) -> CommandContext.sendMessage() -> Network Layer -> Client Feedback Message


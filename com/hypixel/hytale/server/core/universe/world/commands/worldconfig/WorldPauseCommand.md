---
description: Architectural reference for WorldPauseCommand
---

# WorldPauseCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.worldconfig
**Type:** Transient

## Definition
```java
// Signature
public class WorldPauseCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldPauseCommand is a concrete implementation of the Command Pattern, designed to integrate seamlessly into the server's command processing framework. It encapsulates the specific logic required to toggle the paused state of a game world.

Architecturally, this class serves as a leaf node in the command system. It is registered under the alias "pause" and is invoked by the central command dispatcher when a matching user input is received. Its primary responsibility is not to manage the pause state itself, but to act as a safe, context-aware gateway to the World object's state-mutating methods.

By extending AbstractWorldCommand, it leverages a framework that automatically injects critical context, such as the target World and its associated EntityStore, into the execution method. This decouples the command's logic from the complex machinery of context retrieval and validation.

## Lifecycle & Ownership
- **Creation:** A single instance of WorldPauseCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is discovered and registered automatically, making it available for the server's entire operational lifetime.
- **Scope:** The object instance is a long-lived singleton, managed by the command registry. It persists until the server shuts down. The execution context provided to its methods, however, is ephemeral and valid only for a single command invocation.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server's command registry is cleared during the shutdown process.

## Internal State & Concurrency
- **State:** This class is fundamentally **stateless**. It contains no mutable instance fields. All operations are performed on stateful objects, like World and CommandContext, that are passed as method parameters. The static message fields are immutable constants.
- **Thread Safety:** The class is inherently **thread-safe** due to its stateless design. However, the command system guarantees that the execute method is invoked on the main server thread. This prevents race conditions with other game logic that interacts with the World object. Direct invocation from other threads is unsupported and will lead to state corruption.

## API Surface
The public API is minimal and defined by its role as a command handler. Direct interaction with this class outside the command framework is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the pause/unpause logic. Validates game state (single-player) and mutates the World's paused flag. Sends feedback to the user via the CommandContext. |

## Integration Patterns

### Standard Usage
Developers and server administrators do not interact with this class directly in code. The standard interaction is through the game console or chat by a player with sufficient permissions.

```
// User input in the game console
/pause
```
The command system routes this input to the singleton instance of WorldPauseCommand for execution.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WorldPauseCommand()`. The command framework is solely responsible for the lifecycle of command objects. Manual instantiation will result in a disconnected object that is not registered with the command dispatcher.
- **Manual Execution:** Do not obtain the command from a registry and call its execute method directly. This bypasses the framework's critical pre-processing steps, including permission checks, argument parsing, and context validation, which can lead to server instability or security vulnerabilities.
- **Stateful Implementation:** Modifying this class to hold instance-level state is a severe violation of its design. Commands must be stateless and re-entrant.

## Data Pipeline
As a command, this class participates in a control flow rather than a data processing pipeline. It receives a command context and triggers a state change in a core system.

> Flow:
> User Input (`/pause`) -> Network Layer -> Command Dispatcher -> **WorldPauseCommand.execute()** -> World.setPaused() -> Message Bus -> User Feedback Packet


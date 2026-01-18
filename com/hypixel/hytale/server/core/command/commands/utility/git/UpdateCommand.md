---
description: Architectural reference for UpdateCommand
---

# UpdateCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.git
**Type:** Component

## Definition
```java
// Signature
public class UpdateCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The UpdateCommand class implements the *Composite* design pattern to act as a container for a group of related sub-commands. It does not define any executable logic itself; its sole responsibility is to provide a top-level command namespace, in this case **update**, under which other, more specific commands can be organized.

This class serves as a routing node within the server's Command System. When a user executes a command like `/update assets`, the Command System first identifies UpdateCommand as the handler for the `update` token. UpdateCommand then delegates the remainder of the command (`assets`) to its registered sub-commands, ultimately invoking the correct child command, UpdateAssetsCommand.

This architectural choice promotes a clean, hierarchical, and discoverable command-line interface for server administrators, preventing the pollution of the global command namespace.

### Lifecycle & Ownership
- **Creation:** An instance of UpdateCommand is created once during the server's bootstrap phase. The core CommandSystem is responsible for discovering and instantiating all command classes, including this one, to build its command registry.
- **Scope:** The object's lifecycle is bound to the server session. It persists in the CommandSystem's registry for as long as the server is running.
- **Destruction:** The instance is eligible for garbage collection upon server shutdown when the CommandSystem and its associated registries are cleared.

## Internal State & Concurrency
- **State:** The internal state of UpdateCommand is effectively immutable after construction. The constructor populates a list of sub-commands inherited from AbstractCommandCollection. This collection is not modified during the component's lifetime.
- **Thread Safety:** This class is inherently thread-safe. As its state is established at creation and never mutated, it can be safely accessed by multiple threads. The broader Command System is responsible for managing the concurrency of command *execution*, ensuring that commands are processed in a safe and orderly manner, but the UpdateCommand object itself poses no concurrency risks.

## API Surface
The public contract of this class is fulfilled through its constructor and the interface defined by its parent, AbstractCommandCollection. It has no unique public methods. Its primary interaction with the engine is its instantiation and registration by the CommandSystem. The methods for adding sub-commands are used internally during its own construction.

## Integration Patterns

### Standard Usage
Developers do not interact with instances of this class directly. The CommandSystem automatically discovers and registers it. The standard "usage" is from the perspective of a server administrator entering the command in the console.

The registration process, handled by the engine, is conceptually similar to the following:

```java
// Conceptual example of engine bootstrap
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.register(new UpdateCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not instantiate this class manually via `new UpdateCommand()` for any purpose other than registration with the primary CommandSystem. It provides no functionality outside of that context.
- **State Mutation:** Do not attempt to add or remove sub-commands from this collection after its initial construction. The command registry is designed to be immutable after the server starts.

## Data Pipeline
UpdateCommand acts as a dispatcher in the command processing pipeline. It receives control from the CommandSystem and forwards it to a more specific sub-command based on the input tokens.

> Flow:
> Server Console Input (`/update assets`) -> Command Parser -> CommandSystem (resolves `update`) -> **UpdateCommand** (resolves `assets`) -> UpdateAssetsCommand (executes logic) -> Command Result -> Console Output


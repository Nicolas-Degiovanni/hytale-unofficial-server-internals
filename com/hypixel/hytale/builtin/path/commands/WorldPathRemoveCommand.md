---
description: Architectural reference for WorldPathRemoveCommand
---

# WorldPathRemoveCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class WorldPathRemoveCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldPathRemoveCommand class is a concrete implementation of the Command Pattern, designed to integrate seamlessly into the server's command processing system. It represents a single, atomic server operation: the removal of a named path configuration from a specific world.

As a subclass of AbstractWorldCommand, it inherits the necessary infrastructure to be discovered and managed by the server's central command dispatcher. Its primary responsibility is to define its command signature (the name "remove" and a required string argument "name") and to encapsulate the logic for executing the removal. This class acts as a transactional bridge between user input and world state modification, ensuring that administrative actions are handled through a controlled and validated pathway.

## Lifecycle & Ownership
- **Creation:** A single instance is created by the command registration system during server bootstrap. The system scans for command definitions and registers them in a global command tree or map.
- **Scope:** The object instance is a long-lived singleton for the duration of the server's runtime, held as a reference within the command dispatcher. However, its execution context is ephemeral, created anew for each command invocation.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down or the parent command module is unloaded.

## Internal State & Concurrency
- **State:** This class is effectively stateless and immutable after construction. Its fields, such as nameArg, define the command's structure but do not hold data that changes during runtime. The state it modifies is entirely external, belonging to the World and WorldPathConfig objects passed into the execute method.
- **Thread Safety:** This class is **not thread-safe** and must not be treated as such. All command execution is expected to be marshaled onto the main server thread by the command system. Direct invocation of the execute method from an asynchronous task or worker thread will lead to world state corruption and severe server instability. The underlying World object is not designed for concurrent modification.

## API Surface
The public contract is minimal, intended for use only by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the path removal logic. Throws exceptions if context is invalid. Assumes O(1) for map-based removal in WorldPathConfig. |

## Integration Patterns

### Standard Usage
A developer or user does not interact with this class directly. The system invokes it implicitly based on user input. The command is executed by the server's command dispatcher after parsing a command string entered by a player or the console.

**Conceptual Invocation (User Perspective):**
```
/worldpath remove "MyFirstPath"
```

The above user command triggers the server's command processing pipeline, which identifies and calls the execute method on the registered WorldPathRemoveCommand instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually using `new WorldPathRemoveCommand()`. Doing so bypasses the command registration and parsing system, rendering the command non-functional.
- **Manual Execution:** Do not call the execute method directly. This circumvents critical framework services, including permission checks, argument validation, and thread safety guarantees provided by the command dispatcher.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a feedback message, modifying world state in the process.

> Flow:
> User Command String -> Server Command Dispatcher -> Argument Parser -> **WorldPathRemoveCommand.execute()** -> World.getWorldPathConfig().removePath() -> CommandContext.sendMessage() -> User Feedback Message


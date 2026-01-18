---
description: Architectural reference for InstanceEditNewCommand
---

# InstanceEditNewCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Transient

## Definition
```java
// Signature
public class InstanceEditNewCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The InstanceEditNewCommand is a concrete implementation of the Command Pattern, designed to handle server-side administrative tasks. It resides within the server's command processing system and is responsible for creating new game "instance" configurations on the file system.

Architecturally, this class serves as a bridge between the command system and the asset management layer. By extending AbstractAsyncCommand, it signals to the engine that its execution may involve blocking operations, specifically file system I/O. The engine's command dispatcher executes this command on a worker thread, ensuring that the main server loop remains unblocked and responsive. Its primary function is to scaffold the necessary directory structure and default configuration files (instance.bson) required for a new game instance, a foundational task for server operators.

## Lifecycle & Ownership
- **Creation:** This command object is instantiated once by the server's command registration service during the bootstrap sequence. The system likely scans the classpath for command implementations and registers them for later invocation.
- **Scope:** A single instance of InstanceEditNewCommand persists for the entire runtime of the server. It is stateless from an execution perspective; all per-request data is encapsulated within the CommandContext object passed during execution.
- **Destruction:** The object is dereferenced and eligible for garbage collection when the server shuts down and its central command registry is cleared.

## Internal State & Concurrency
- **State:** The class holds immutable state defining its command arguments (instanceNameArg, packName). These fields are configured in the constructor and do not change during the object's lifetime. All mutable state related to a specific command execution is passed in via the CommandContext parameter.
- **Thread Safety:** This class is thread-safe. Its own state is immutable post-construction. The executeAsync method is designed to be executed off the main server thread. It safely interacts with global singletons like AssetModule, which are assumed to be thread-safe themselves. The method's logic is self-contained and does not produce side effects that would interfere with concurrent executions for different users.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | I/O Bound | Asynchronously creates a new instance configuration. Validates that the target AssetPack is mutable, creates the required directories, and writes a default WorldConfig to an instance.bson file. Reports success or I/O errors to the caller via the CommandContext. |

## Integration Patterns

### Standard Usage
This class is not designed for direct programmatic invocation. It is exclusively managed and executed by the server's command dispatcher in response to user input from the console or an in-game player with sufficient permissions.

The typical invocation is via the server console:
```text
// Creates a new instance named "dungeon-1" in the default asset pack
instance edit new dungeon-1

// Creates a new instance named "pvp-arena" in a specific asset pack
instance edit new pvp-arena pack:custom_assets
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new InstanceEditNewCommand()`. The command will not be registered with the server and will have no effect. The command system handles the lifecycle.
- **Blocking on the Future:** If you were to somehow invoke this method programmatically, calling `get()` on the returned CompletableFuture from the main server thread would freeze the server until the file I/O is complete, defeating the purpose of the asynchronous design.

## Data Pipeline
The flow of data for this command begins with user input and ends with a file system modification and a feedback message.

> Flow:
> User Input String -> Command Parser -> Command Dispatcher -> **InstanceEditNewCommand.executeAsync** -> AssetModule -> Filesystem I/O -> WorldConfig.save -> CommandContext.sendMessage -> User Feedback Message


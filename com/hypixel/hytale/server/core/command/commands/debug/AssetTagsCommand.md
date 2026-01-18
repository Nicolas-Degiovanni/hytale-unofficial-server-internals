---
description: Architectural reference for AssetTagsCommand
---

# AssetTagsCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Transient

## Definition
```java
// Signature
public class AssetTagsCommand extends CommandBase {
```

## Architecture & Concepts
The AssetTagsCommand is a server-side diagnostic tool that provides a runtime interface for querying the Hytale asset system. It is a concrete implementation within the server's Command System, designed to be invoked by administrators or developers via the server console.

This command's primary architectural role is to act as a read-only bridge between a user-facing command and the low-level **AssetRegistry**. While assets are typically retrieved by their unique key or numerical ID, this command facilitates a reverse lookup, allowing introspection of the asset database based on metadata tags. This is critical for debugging content packs, verifying asset configurations, and understanding the composition of asset groups without needing to inspect the source files directly.

It operates by iterating through all registered AssetStores, matching a specified asset type, and then querying the underlying AssetMap for all entries associated with a given tag.

## Lifecycle & Ownership
- **Creation:** A single instance of AssetTagsCommand is instantiated by the CommandSystem during server bootstrap. The system discovers all subclasses of CommandBase and registers them for runtime invocation.
- **Scope:** The command object itself is a long-lived singleton, persisting for the entire server session. However, its execution is ephemeral; the core logic in executeSync is invoked only when the command is run, and all context is passed in via the CommandContext parameter.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the CommandSystem is dismantled.

## Internal State & Concurrency
- **State:** The class contains effectively immutable state. The fields classArg and tagArg are final RequiredArg instances that define the command's argument signature. They are configured once in the constructor and never change. The command itself holds no mutable state and does not cache results between executions.
- **Thread Safety:** This command is **not thread-safe** and must be invoked on the main server thread. The CommandSystem guarantees that executeSync is called synchronously within the primary server tick loop. This is a critical design constraint, as the command performs read operations on the AssetRegistry, which is not designed for safe concurrent access from arbitrary threads.

**WARNING:** Attempting to invoke executeSync from an asynchronous task or a different thread will lead to race conditions and likely corrupt the state of the AssetRegistry, causing server instability or crashes.

## API Surface
The public contract is exclusively defined by its inheritance from CommandBase and is intended for invocation by the CommandSystem only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Executes the tag query. N is the number of registered asset types. Throws exceptions if arguments are missing, which is handled by the CommandSystem. |

## Integration Patterns

### Standard Usage
A developer does not directly instantiate or invoke this class. Instead, it is automatically registered with the CommandSystem. An administrator or developer uses it via the server console.

The conceptual flow of invocation by the system is as follows:

```java
// PSEUDO-CODE: How the CommandSystem invokes this command
CommandSystem commandSystem = server.getCommandSystem();
CommandContext context = commandSystem.createContextFor("/assets tags Block stone");

// The system finds the 'AssetTagsCommand' registration and calls it
CommandBase command = commandSystem.findCommand("assets", "tags");
command.execute(context); // This eventually calls executeSync
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AssetTagsCommand()`. The CommandSystem is responsible for the lifecycle of all command objects. Manually creating an instance will result in a non-functional command that is not registered to handle any input.
- **Direct Invocation:** Never call `executeSync` directly. This bypasses critical infrastructure provided by the CommandSystem, including argument parsing, permission checks, and context setup. All interaction must be routed through the central command dispatcher.
- **Stateful Implementation:** Do not modify this class to store state between executions. Commands are designed to be stateless request handlers.

## Data Pipeline
The command acts as a handler in a data flow that begins with user input and ends with a formatted message sent back to the user.

> Flow:
> User Console Input (`/assets tags ...`) -> Server Network Listener -> Command Parser -> CommandSystem Dispatcher -> **AssetTagsCommand.executeSync** -> AssetRegistry Query -> Result Set (Keys/Indexes) -> **AssetTagsCommand** -> Message & MessageFormat -> CommandContext.sendMessage -> Server Network Emitter -> User Console Output


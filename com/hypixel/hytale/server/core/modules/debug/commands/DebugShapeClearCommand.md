---
description: Architectural reference for DebugShapeClearCommand
---

# DebugShapeClearCommand

**Package:** com.hypixel.hytale.server.core.modules.debug.commands
**Type:** Transient

## Definition
```java
// Signature
public class DebugShapeClearCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DebugShapeClearCommand is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central CommandSystem. It encapsulates a single, specific server-side action: clearing all active debug shapes from a given world instance.

This class acts as a leaf node within the server's command tree, likely registered under a parent command such as *debug* or *shape*. Its primary architectural role is to decouple the command parsing and dispatching logic from the underlying debug visualization system. It achieves this by delegating its core operation to the DebugUtils utility class, ensuring a clean separation of concerns. The command itself is responsible only for receiving the execution context and triggering the appropriate utility function.

As it extends AbstractPlayerCommand, it is implicitly tied to an execution context that requires a valid player entity.

## Lifecycle & Ownership
- **Creation:** A single instance is created by the CommandSystem or a related registry during server bootstrap or when the debug module is loaded. It is then registered under the name "clear" within its parent command's namespace.
- **Scope:** The command object instance is a singleton managed by the CommandSystem and persists for the entire server session. The execution context provided to the *execute* method, however, is transient and specific to a single command invocation.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down or the parent module is unloaded.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable instance fields. The only field, MESSAGE_COMMANDS_DEBUG_SHAPE_CLEAR_SUCCESS, is a static final constant, making it immutable and shared across all potential invocations. All state required for execution, such as the target World, is passed as method parameters.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, command execution is expected to be serialized and performed on the main server thread by the CommandSystem. This is a critical assumption to prevent race conditions when the `DebugUtils.clear` method modifies the state of the World object. Direct invocation from an asynchronous thread is not supported and will lead to world state corruption.

## API Surface
The public contract is minimal, designed for invocation by the command framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DebugShapeClearCommand() | constructor | O(1) | Initializes the command with its name and description key. Intended for framework use only. |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the command logic. Complexity is dependent on the number of debug shapes (N) stored in the world. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly from code. It is invoked by a player or the server console through the chat or command-line interface. The CommandSystem is responsible for parsing the input, locating this command, and invoking it with the correct context.

*Example Player Input:*
```
/debug shape clear
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DebugShapeClearCommand()`. The server's CommandSystem manages the lifecycle of all command objects. Manually creating an instance will result in an object that is not registered and cannot be executed.
- **Manual Invocation:** Avoid calling the `execute` method directly. Bypassing the CommandSystem means skipping critical steps like permission checks, context validation, and argument parsing, which can lead to instability or security vulnerabilities.

## Data Pipeline
The data flow for this command is initiated by external user input and results in a modification of the world state.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> CommandSystem Dispatcher -> **DebugShapeClearCommand.execute()** -> DebugUtils.clear() -> World State Mutation -> CommandContext.sendMessage() -> Network Packet -> Player Chat Output


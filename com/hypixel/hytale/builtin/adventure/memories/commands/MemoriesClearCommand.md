---
description: Architectural reference for MemoriesClearCommand
---

# MemoriesClearCommand

**Package:** com.hypixel.hytale.builtin.adventure.memories.commands
**Type:** Singleton

## Definition
```java
// Signature
public class MemoriesClearCommand extends CommandBase {
```

## Architecture & Concepts
The MemoriesClearCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's core command processing system. Its sole responsibility is to provide an administrative entry point for clearing all data managed by the MemoriesPlugin.

This class acts as a thin, stateless adapter. It translates a server command, issued by a player or the console, into a direct method call on the MemoriesPlugin service. This decouples the command system, which handles parsing and permissions, from the business logic of the memory management system itself. By extending CommandBase, it adheres to the server's contract for command registration and execution.

## Lifecycle & Ownership
- **Creation:** A single instance of MemoriesClearCommand is instantiated by the server's CommandSystem during the registration phase of its parent, MemoriesPlugin. The command is registered under the name "clear" within the parent "memories" command group.
- **Scope:** The instance is a singleton managed by the CommandSystem's registry. It persists for the entire lifecycle of the server, or until the MemoriesPlugin is unloaded.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down or the parent plugin is disabled, at which point the CommandSystem clears its registration.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. Its only member field is a static final Message object, which is a thread-safe, pre-computed reference. All stateful operations are delegated to the MemoriesPlugin singleton.
- **Thread Safety:** The primary method, executeSync, is explicitly designed to run on the main server thread. The name itself serves as a contract. Invoking this method from any other thread is a critical violation of the server's threading model and will lead to race conditions and data corruption within the MemoriesPlugin. The class is inherently thread-safe, but its *downstream interactions* are not.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Clears all recorded memories. Complexity is O(N) where N is the number of memories stored in the MemoriesPlugin. Sends a confirmation message to the command issuer via the provided context. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked automatically by the server's command dispatcher when an authorized entity executes the corresponding command.

*Example command execution:*
```
/memories clear
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new MemoriesClearCommand()`. The command system is responsible for the lifecycle of all registered commands. Manual instantiation creates an unmanaged object that will not be reachable by the command parser.
- **Direct Invocation:** Avoid calling the `executeSync` method directly. Doing so bypasses the server's critical permission checks, argument parsing, and context setup, potentially leaving the system in an inconsistent state. Always use the server's command dispatch service.

## Data Pipeline
MemoriesClearCommand is part of a control flow rather than a data processing pipeline. It receives a command signal and translates it into an action on a different system.

> Flow:
> Player/Console Input (`/memories clear`) -> Server Command Parser -> Command Dispatcher -> **MemoriesClearCommand.executeSync()** -> MemoriesPlugin.clearRecordedMemories() -> CommandContext.sendMessage() -> Network Layer -> Player Client (receives success message)


---
description: Architectural reference for WorldListCommand
---

# WorldListCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldListCommand extends CommandBase {
```

## Architecture & Concepts
The WorldListCommand is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central Command System. Its sole responsibility is to provide a user-facing mechanism for querying and listing all currently active worlds within the server instance.

Architecturally, this class serves as a read-only bridge between a command-issuing entity (such as a player or the server console) and the core state of the server, represented by the Universe singleton. It retrieves the set of world names, formats them into a translatable, user-friendly list using the MessageFormat utility, and dispatches the result back to the original sender. It does not perform any state modification.

The class inherits from CommandBase, which provides the necessary contract for integration with the command dispatcher, including argument parsing, permission handling, and execution lifecycle hooks.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the Command System during the server's bootstrap phase. The system scans for command implementations and registers a singleton instance of each for later dispatch.
- **Scope:** Application-scoped. The single instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only upon server shutdown when the Command System clears its registry.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields and does not cache data between invocations. All required information is sourced directly from the Universe singleton and the provided CommandContext during execution.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The framework guarantees that the primary logic method, executeSync, is invoked on the main server thread. This design prevents race conditions when accessing shared game state, such as the world list within the Universe. Direct, multi-threaded invocation is an unsupported and dangerous pattern.

## API Surface
The public API is minimal and intended for framework use only. Developers do not interact with this class's methods directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(N) | Executes the command logic. Retrieves N worlds from the Universe, formats them, and sends the result to the command sender. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is automatically registered by the server's command handler. A user with appropriate permissions invokes it by typing the command into the chat or console.

**User Invocation Example:**
```
/worlds list
```
or its alias:
```
/worlds ls
```

**Expected Server Response:**
```
Worlds (2): world, another_world
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually via `new WorldListCommand()`. The command system manages its lifecycle. Manual instantiation bypasses critical infrastructure such as permission checks and context injection.
- **Manual Execution:** Do not call the executeSync method directly. This circumvents the command dispatcher and its thread safety guarantees, creating a high risk of concurrency exceptions when accessing game state from an improper thread.
- **Stateful Implementation:** Do not modify this class to store state in instance variables. Commands must be stateless to ensure they are re-entrant and behave predictably across all invocations.

## Data Pipeline
The flow of data for a typical invocation is unidirectional, originating from user input and terminating as a message sent back to that user.

> Flow:
> User Input (`/worlds list`) -> Network Layer -> Command Parser -> **WorldListCommand.executeSync()** -> Universe State Query -> MessageFormat -> Command Sender -> Network Layer -> Client Chat UI


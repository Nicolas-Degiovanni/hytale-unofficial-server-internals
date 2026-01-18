---
description: Architectural reference for BlockSetCommand
---

# BlockSetCommand

**Package:** com.hypixel.hytale.server.core.modules.blockset.commands
**Type:** Transient Component

## Definition
```java
// Signature
public class BlockSetCommand extends CommandBase {
```

## Architecture & Concepts
The BlockSetCommand class is a concrete implementation of the Command Pattern, designed to be registered with and executed by the server's central CommandSystem. Its primary function is to serve as a user-facing diagnostic and administrative tool for inspecting the contents of various block sets managed by the BlockSetModule.

This command acts as a read-only interface to two key data sources:
1.  **BlockSetModule:** The live, in-memory cache of block set collections.
2.  **Static Asset Maps:** The global, immutable registries for BlockSet and BlockType definitions loaded from game assets.

Architecturally, it decouples the low-level data management of the BlockSetModule from the high-level user interaction of the command console. It translates user-provided string identifiers into internal asset indices, performs lookups, and formats the results into human-readable messages.

## Lifecycle & Ownership
- **Creation:** An instance of BlockSetCommand is created by its parent BlockSetModule, typically during the module's own initialization phase. The module injects a reference to itself into the command's constructor.
- **Scope:** The command object is designed to be long-lived. Once instantiated and registered with the CommandSystem, it persists for the entire server session, or until the parent BlockSetModule is unloaded.
- **Destruction:** The object is eligible for garbage collection when the CommandSystem clears its registration entry, which typically occurs when the owning BlockSetModule is shut down. There is no explicit destruction logic within the class itself.

## Internal State & Concurrency
- **State:** The internal state of a BlockSetCommand instance is **effectively immutable** after construction. It holds final references to the BlockSetModule and its argument definitions. It does not cache query results or modify its own state during execution. All data is retrieved fresh from the module or static asset maps upon each invocation.

- **Thread Safety:** This class is **not thread-safe** and must not be invoked from arbitrary threads. The `executeSync` method name is a strong convention indicating that the CommandSystem guarantees its execution on the main server thread. This design avoids the need for internal locking by relying on a single-threaded execution model for command processing, which prevents race conditions when accessing shared module state or asset maps.

## API Surface
The primary contract is fulfilled by overriding the `executeSync` method from its parent, CommandBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BlockSetCommand(blockSetModule) | constructor | O(1) | Constructs the command. Requires a non-null BlockSetModule dependency. |
| executeSync(context) | void | O(N log N) | Executes the command logic. If a blockset is specified, complexity is dominated by sorting the N block names. If no arguments are given, complexity is O(M) where M is the total number of registered blocksets. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be instantiated by a module and registered with the server's CommandSystem. The system then dispatches to this command based on user input.

```java
// Example of registration within a module's lifecycle method
// NOTE: 'this' refers to an instance of BlockSetModule.
CommandSystem commandSystem = context.getService(CommandSystem.class);
commandSystem.register(new BlockSetCommand(this));
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call `executeSync` directly. This bypasses the command system's critical infrastructure, including argument parsing, permission checks, and thread safety guarantees. The CommandContext object would be difficult to mock correctly and could lead to unpredictable behavior.
- **Stateful Implementation:** Do not modify this class to store state between executions. Command objects are expected to be stateless handlers. Caching should be delegated to the underlying module or service.
- **Instantiation without Registration:** Creating an instance of BlockSetCommand without registering it with the CommandSystem serves no purpose, as the object will never be used.

## Data Pipeline
The command facilitates a simple query-response data flow, translating a user's text command into a structured list of game data.

> Flow:
> User Input (`/blockset <name>`) -> CommandSystem Parser -> **BlockSetCommand.executeSync** -> BlockSetModule & Static Asset Maps -> MessageFormat -> CommandContext -> Formatted User Output


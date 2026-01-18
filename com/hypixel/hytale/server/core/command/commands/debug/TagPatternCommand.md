---
description: Architectural reference for TagPatternCommand
---

# TagPatternCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Transient

## Definition
```java
// Signature
public class TagPatternCommand extends CommandBase {
```

## Architecture & Concepts
The TagPatternCommand is a concrete implementation of the CommandBase abstraction, designed to serve as a server-side debug utility. It provides a direct interface for administrators and developers to test the logic of asset tags without requiring complex in-game scenarios.

Its primary architectural role is to bridge the server's command system with the asset management system. It declaratively defines its required inputs—a TagPattern asset and a BlockType asset—using the argument parsing framework. Upon execution, it retrieves these resolved assets from the CommandContext, performs a boolean test to see if the block's tags match the pattern, and reports the result back to the command's sender. This provides immediate, low-level feedback on asset configuration.

## Lifecycle & Ownership
- **Creation:** A single instance of TagPatternCommand is created by the server's command registration system during the server bootstrap sequence. It is not instantiated on a per-request basis.
- **Scope:** The object instance is held by the central CommandRegistry and persists for the entire lifetime of the server session.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless from an execution perspective. Its instance fields, such as tagPatternArg and blockTypeArg, are immutable definitions of the command's argument structure, configured once during construction. The core execution logic in executeSync operates exclusively on the transient CommandContext passed into it.
- **Thread Safety:** The class is inherently thread-safe. The `executeSync` method name strongly implies that the command system dispatches it synchronously on the main server thread. All instance fields are read-only after construction, eliminating the risk of data races from within the object itself. No explicit locking mechanisms are required or used.

## API Surface
The public contract is fulfilled by its registration with the command system. The `executeSync` method is the entry point for the command's logic, but it is not intended for direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Executes the tag matching logic. N is the number of tags on the provided BlockType. Retrieves parsed assets from the context, invokes TagPattern.test, and sends a formatted result Message. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically. It is invoked by a user (e.g., a server administrator) through the server console or in-game chat interface.

**Example Console Input:**
```
/tagpattern hytale:stone_pattern hytale:stone
```
This command would test if the BlockType asset identified by *hytale:stone* has tags that match the rules defined in the TagPattern asset *hytale:stone_pattern*.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create an instance using `new TagPatternCommand()`. The command framework is solely responsible for its lifecycle. Manually created instances will not be registered and will be non-functional.
- **Manual Execution:** Never call the `executeSync` method directly. It relies on a fully constructed CommandContext, which is only provided by the server's command dispatcher during a legitimate command invocation. Bypassing the dispatcher will result in NullPointerExceptions or other undefined behavior.

## Data Pipeline
The flow of data for this command begins with user input and ends with a message sent back to the user's client.

> Flow:
> User Input String -> Command Dispatcher -> Argument Parser -> **TagPatternCommand.executeSync** -> Message Factory -> CommandContext.sendMessage -> Network Packet -> Client Chat UI


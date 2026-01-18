---
description: Architectural reference for SetToolHistorySizeCommand
---

# SetToolHistorySizeCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class SetToolHistorySizeCommand extends CommandBase {
```

## Architecture & Concepts
The SetToolHistorySizeCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's central command processing system. Its sole responsibility is to provide a user-facing command, `/setToolHistorySize`, that modifies a configuration value within the BuilderToolsPlugin.

This class acts as a declarative endpoint for user input. It defines its required arguments, their types, and associated validation rules (e.g., RangeValidator) directly within its fields. This allows the core Command System to handle the complex tasks of parsing, type conversion, and input validation before the command's execution logic is ever invoked.

By extending CommandBase, it registers itself with the server's CommandRegistry, making it discoverable and executable by authorized users. The permission check, `setPermissionGroup(GameMode.Creative)`, ensures that this command is only accessible to players in a privileged game mode, effectively namespacing its functionality to the build/creative context.

## Lifecycle & Ownership
- **Creation:** A prototype instance of this class is created by the CommandRegistry during server initialization or when the `builtin.buildertools` plugin is loaded. The registry scans for all subtypes of CommandBase and instantiates them.
- **Scope:** The single prototype instance persists for the entire server session. It is not instantiated per-execution. The `CommandContext` object passed to the `executeSync` method is ephemeral and scoped to a single command invocation.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down or the owning plugin is unloaded.

## Internal State & Concurrency
- **State:** This class is stateless. Its fields, such as `historyLengthArg`, are final definitions that describe the command's structure but do not hold mutable data related to its execution. The state it modifies is external, managed entirely by the BuilderToolsPlugin singleton.
- **Thread Safety:** Not thread-safe. The `executeSync` method is explicitly designed to be called synchronously on the main server thread as part of the game tick loop. Invoking this method from any other thread will lead to race conditions and severe state corruption. The Command System guarantees this synchronous execution contract.

## API Surface
The public contract is primarily defined by its superclass, CommandBase, for integration with the Command System. The core logic is encapsulated in the protected `executeSync` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Modifies the tool history size in the BuilderToolsPlugin and sends a confirmation message to the command issuer. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked programmatically. It is executed by the server's command handler in response to player input.

A player with Creative permissions types the following into the chat console:
```
/setToolHistorySize 100
```
The system then parses this input, validates that 100 is within the allowed range of 10-250, and invokes the `executeSync` method on the registered command instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SetToolHistorySizeCommand()`. The object will not be registered with the command system and will be non-functional.
- **Manual Execution:** Do not call the `executeSync` method directly. Doing so bypasses critical infrastructure, including permission checks, argument validation, and context setup, which can leave the system in an inconsistent state.

## Data Pipeline
The flow for this command is initiated by user input and terminates with a state change and user feedback.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> Argument & Permission Validation -> **SetToolHistorySizeCommand.executeSync** -> BuilderToolsPlugin.setToolHistorySize -> CommandContext.sendMessage -> Network Packet -> Player Chat Display


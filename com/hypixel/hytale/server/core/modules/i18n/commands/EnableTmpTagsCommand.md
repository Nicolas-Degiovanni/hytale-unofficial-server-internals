---
description: Architectural reference for EnableTmpTagsCommand
---

# EnableTmpTagsCommand

**Package:** com.hypixel.hytale.server.core.modules.i18n.commands
**Type:** Transient

## Definition
```java
// Signature
public class EnableTmpTagsCommand extends CommandBase {
```

## Architecture & Concepts
The EnableTmpTagsCommand is a concrete implementation of the Command Pattern, designed for server administration and debugging. It registers itself with the server's central command system under the primary name *toggletmptags* and several aliases.

Its sole architectural function is to act as a user-facing trigger to mutate a single boolean flag within the global HytaleServerConfig object. This flag, *displayTmpTagsInStrings*, controls whether the server's internationalization (i18n) system renders raw translation keys (e.g., *item.sword.name*) or their translated values (e.g., "Sword").

This command is not a gameplay feature but a diagnostic tool for developers and content creators to verify that text elements are correctly wired into the localization pipeline. Its direct access to HytaleServer and HytaleServerConfig signifies its role as a trusted, core administrative utility.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the CommandSystem during server bootstrap. The system scans for classes extending CommandBase and instantiates them for registration.
- **Scope:** The command object is stateless and its instance persists for the entire server session, held within the CommandSystem's internal registry.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only upon server shutdown when the CommandSystem is dismantled.

## Internal State & Concurrency
- **State:** This class is **entirely stateless**. It contains no mutable member variables and all state it reads and modifies is external, located within the HytaleServerConfig singleton.
- **Thread Safety:** The method name *executeSync* is a critical convention, indicating that the CommandSystem guarantees its execution on the main server thread. The implementation is therefore not thread-safe by design and relies on the caller to provide a synchronized context.

**WARNING:** Invoking executeSync from any thread other than the main server thread will lead to race conditions when accessing HytaleServerConfig and will likely corrupt server state.

## API Surface
The public contract is minimal, primarily defined by its constructor for registration and the execution method for invocation by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EnableTmpTagsCommand() | constructor | O(1) | Registers the command's primary name, aliases, and description key with its superclass. |
| executeSync(CommandContext) | void | O(1) | Atomically toggles the configuration flag and sends a confirmation message to the command source. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked automatically by the server's command parser in response to user input from the console or an in-game chat.

A developer or server administrator would use it as follows in the server console:
```
toggletmptags
```

The system then dispatches this to the registered instance. A conceptual view of the dispatch logic is:

```java
// Conceptual example of how the CommandSystem invokes this.
// Do not replicate this pattern.

CommandContext context = createFromConsoleInput("/toggletmptags");
CommandBase command = commandRegistry.find("toggletmptags");

if (command instanceof EnableTmpTagsCommand) {
    command.executeSync(context);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EnableTmpTagsCommand()`. The command system handles the lifecycle. Creating your own instance will result in an un-registered command that does nothing.
- **Direct Invocation:** Do not call `command.executeSync()` directly. This bypasses the CommandSystem's middleware, which may include permission checks, cooldowns, and logging. Always dispatch commands through the appropriate server API.

## Data Pipeline
The flow for this command is initiated by external user input and results in a mutation of global server state and a feedback message.

> Flow:
> User Input (e.g., `/tmptags`) -> Network Layer -> Command Parser -> CommandSystem Dispatcher -> **EnableTmpTagsCommand.executeSync()** -> HytaleServerConfig (State Mutation) -> CommandContext (Message Send) -> Network Layer -> Client UI (Feedback)


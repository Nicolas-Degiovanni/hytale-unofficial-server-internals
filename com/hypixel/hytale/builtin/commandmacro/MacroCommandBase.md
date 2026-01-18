---
description: Architectural reference for MacroCommandBase
---

# MacroCommandBase

**Package:** com.hypixel.hytale.builtin.commandmacro
**Type:** Base Class

## Definition
```java
// Signature
public class MacroCommandBase extends AbstractAsyncCommand {
```

## Architecture & Concepts
The MacroCommandBase class is a foundational component of the server's command system, designed to create powerful, user-defined "macro" commands. It functions as a translation and execution engine, converting a single, high-level command invocation into a sequence of one or more lower-level server commands.

Architecturally, this class decouples the command's public interface from its implementation. It allows developers and server administrators to define complex command chains and shortcuts via configuration rather than writing new Java code. The core responsibility of this class is to parse a macro definition at startup, and at runtime, to perform parameter substitution and dispatch the resulting commands back into the CommandManager for execution.

The system relies on a simple but effective token replacement mechanism, where arguments provided to the macro are injected into predefined command string templates. This makes it a powerful tool for automating repetitive tasks and simplifying complex administrative workflows.

## Lifecycle & Ownership
- **Creation:** Instances of MacroCommandBase, or its subclasses, are created during the server's bootstrap phase. Typically, a command registration system reads macro definitions from configuration files (e.g., JSON) and instantiates this class for each macro to be registered with the CommandManager. The entire state and logic of the macro is defined and validated within the constructor.
- **Scope:** An instance of MacroCommandBase persists for the entire server session. Once registered with the CommandManager, it remains available for execution until the server shuts down.
- **Destruction:** The object is eligible for garbage collection when the server shuts down and the CommandManager clears its command registry, releasing all references to it. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** This class is stateful, containing the complete "compiled" definition of a macro command. This includes its argument specifications (in the *arguments* map), the command templates, and the parsed replacement tokens (in the *commandReplacements* list).
    - **Warning:** The internal state is designed to be **effectively immutable** after the constructor completes. All collections are populated and validated once. Any external modification to this state post-construction will lead to undefined and hazardous behavior.
- **Thread Safety:** The class is **thread-safe for execution**. The internal state is read-only after initialization. The primary execution method, executeAsync, operates on the immutable state and the per-call CommandContext object, making it safe for concurrent invocation from multiple threads or players. Command execution is managed via CompletableFuture, ensuring non-blocking, asynchronous operation.

## API Surface
The public contract is primarily defined by its constructor for initialization and the inherited execution flow from AbstractAsyncCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MacroCommandBase(...) | Constructor | O(N * M) | Initializes and validates the entire macro definition. Parses all command strings for replacement tokens. Throws IllegalArgumentException on invalid configuration. |
| executeAsync(context) | CompletableFuture | O(C * R) | **Protected Method.** Expands the macro by substituting arguments from the context into command templates and dispatches them for execution. |

*Complexity Note: N is the number of command templates, M is the average number of tokens per template. C is the number of commands to expand, R is the number of replacements per command.*

## Integration Patterns

### Standard Usage
MacroCommandBase is not intended for direct, frequent instantiation. It is instantiated once per macro definition and registered with the server's CommandManager.

```java
// A hypothetical factory reads a configuration and registers the macro.
// This process typically happens once at server startup.

// 1. Define parameters for the macro
MacroCommandParameter nameParam = new MacroCommandParameter("player", "The target player.", ...);
MacroCommandParameter reasonParam = new MacroCommandParameter("reason", "The reason for the kick.", ...);

// 2. Define the command templates to be executed
String[] commandsToRun = {
    "say Kicking player {player} for reason: {reason}",
    "kick {player} {reason}"
};

// 3. Instantiate and register the macro command
MacroCommandBase kickMacro = new MacroCommandBase(
    "kickplus",
    new String[]{"k+"},
    "Kicks a player and announces it.",
    new MacroCommandParameter[]{nameParam, reasonParam},
    commandsToRun
);

CommandManager.get().register(kickMacro);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not attempt to modify the internal collections of a MacroCommandBase instance after it has been constructed. The object is not designed for runtime reconfiguration.
- **Recursive Invocation:** Defining a macro that calls itself, either directly or indirectly, can cause an infinite loop and a StackOverflowError. The CommandManager may not have sufficient guards to prevent this. For example, do not create a macro named "kick" that expands to execute the command "kick {player}".
- **Overly Complex Templates:** The replacement engine is for simple string substitution. Avoid embedding complex logic or expecting shell-like evaluation within the command templates.

## Data Pipeline
The flow of data during a macro's execution is a re-entrant loop through the server's command system.

> Flow:
> Player Command Input -> CommandManager -> Argument Parser -> CommandContext -> **MacroCommandBase.executeAsync** -> Internal String Substitution -> A List of New Command Strings -> CommandManager.handleCommand (re-entrant) -> Final Target Command Execution


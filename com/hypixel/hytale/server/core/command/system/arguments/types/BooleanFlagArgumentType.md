---
description: Architectural reference for BooleanFlagArgumentType
---

# BooleanFlagArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class BooleanFlagArgumentType extends ArgumentType<Boolean> {
```

## Architecture & Concepts
The BooleanFlagArgumentType is a specialized, stateless implementation of the ArgumentType contract within the server's command parsing system. Its sole purpose is to represent a command argument that functions as a binary "flag".

Architecturally, it embodies the principle of presence-based input. Unlike other argument types that consume and interpret subsequent string tokens (e.g., IntegerArgumentType, StringArgumentType), this class's logic is triggered merely by its association with a token. The presence of the argument name in the command string implies a value of **true**. It consumes zero subsequent tokens from the input stream.

This component is a terminal node in the parsing strategy, providing a highly efficient, O(1) mechanism for handling simple on/off options in server commands, such as `/teleport --silent` or `/gamemode creative --force`.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by the command system's registration logic. A new instance is typically created for each command definition that requires a flag-style argument. It is not a singleton.
- **Scope:** Short-lived. The object's lifetime is typically scoped to the server's bootstrap phase during command registration. The registered instance may then be used for the duration of the server session.
- **Destruction:** The object is eligible for garbage collection once the command registry itself is destroyed, typically during server shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is constant. The localization keys passed to its super-constructor are the only configured data, and they are immutable.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, a single instance can be safely used by multiple threads concurrently to parse commands without any risk of race conditions or data corruption. No synchronization primitives are necessary.

## API Surface
The public contract is minimal and defined by its parent, ArgumentType.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(input, parseResult) | Boolean | O(1) | Unconditionally returns Boolean.TRUE. The input parameters are ignored, as the presence of the argument is sufficient to determine the result. |

## Integration Patterns

### Standard Usage
This type is not intended to be invoked directly. It is designed to be supplied to a command builder or registration service during server initialization to define the behavior of a command argument.

```java
// Example of declarative command definition
// NOTE: This is a hypothetical builder API for illustration.
CommandBuilder.create("kick")
    .withArgument("player", new PlayerArgumentType())
    .withArgument("force", new BooleanFlagArgumentType()) // Correct usage
    .executes(context -> {
        Player target = context.getArgument("player");
        boolean isForced = context.getArgument("force"); // Will be 'true' if --force was present

        if (isForced) {
            // ... logic for a forced kick
        }
    });
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Parsing:** Do not create an instance of this class to manually parse a value. The command system handles its lifecycle and invocation.
- **Misuse for Valued Booleans:** This class cannot parse string literals like "true" or "false". Using it for an argument where the user is expected to type `/command --option true` will result in incorrect parsing behavior. A different, more complex ArgumentType would be required for that use case.

## Data Pipeline
The flow of data for a command flag is simple and direct, short-circuiting complex parsing logic.

> Flow:
> Raw Command String (`/kick Player1 --force`) -> Command Tokenizer -> Command System identifies `--force` token -> **BooleanFlagArgumentType.parse()** -> Returns `Boolean.TRUE` -> Populates Parsed Command Context -> Command Executor receives `true` for the "force" argument.


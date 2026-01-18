---
description: Architectural reference for MultiArgumentContext
---

# MultiArgumentContext

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class MultiArgumentContext {
```

## Architecture & Concepts
The MultiArgumentContext is a transient, stateful container that acts as a temporary data store during the lifecycle of a single server command invocation. Its primary role is to aggregate the strongly-typed results produced by various ArgumentType parsers.

This class embodies the **Context Object** pattern. It decouples the command parsing stage from the command execution stage. The command parsing system populates this context with typed data (e.g., a Player object, an integer, a string), and the final command handler reads from it. This design prevents the need to pass a long and variable list of parameters to the command executor, providing a clean and extensible interface.

It is a critical component of the server's command system, ensuring that by the time a command's logic is executed, all its required parameters have been successfully parsed, validated, and converted from raw strings into their final object representations.

## Lifecycle & Ownership
- **Creation:** A new MultiArgumentContext instance is created by the core command dispatcher at the beginning of processing a single command string from a client. It is created on a per-request basis.
- **Scope:** The object's lifetime is strictly limited to the duration of one command execution. It exists only to collect argument data for that single invocation.
- **Destruction:** The object is eligible for garbage collection immediately after the command handler has finished execution. There are no external references to it beyond the scope of the command processing pipeline.

## Internal State & Concurrency
- **State:** The MultiArgumentContext is fundamentally **mutable**. Its internal state is the `parsedArguments` map, which is populated incrementally as each argument in a command string is parsed. The state is not intended to be read until it has been fully populated by the command parsing system.
- **Thread Safety:** This class is **not thread-safe**. The underlying map, `Object2ObjectOpenHashMap`, is unsynchronized. The entire command parsing and execution pipeline is expected to operate on a single thread.

**WARNING:** Concurrent calls to `registerArgumentValues` or accessing the context from multiple threads will result in a corrupted state and unpredictable behavior, including potential `ConcurrentModificationException`. Do not share instances of this class across threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerArgumentValues(ArgumentType, String[], ParseResult) | void | O(1) avg. | Populates the context. Delegates parsing to the provided ArgumentType and stores the result, keyed by the type itself. |
| get(ArgumentType) | DataType | O(1) avg. | Retrieves a previously parsed and stored argument value. Returns null if the argument was not parsed. |

## Integration Patterns

### Standard Usage
A command executor receives a fully populated MultiArgumentContext and uses it to retrieve its required parameters in a type-safe manner.

```java
// Inside a command's execute method
public void execute(CommandSource source, MultiArgumentContext context) {
    // Retrieve the parsed arguments using the same ArgumentType objects
    // that defined them.
    Player targetPlayer = context.get(PlayerArgument.player());
    int amount = context.get(IntegerArgument.integer());

    if (targetPlayer != null && amount > 0) {
        targetPlayer.giveItem(Items.DIAMOND, amount);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Never re-use a MultiArgumentContext instance for a subsequent command execution. Its state is specific to a single invocation and must be discarded.
- **External Modification:** Do not attempt to retrieve the context and add values to it outside of the designated command parsing pipeline. The integrity of the context depends on the controlled parsing sequence.
- **Premature Reading:** Do not attempt to `get` a value before the parsing stage for that argument is complete. The command system guarantees the context is fully populated before passing it to the execution logic.

## Data Pipeline
The MultiArgumentContext serves as a staging area for data as it is transformed from raw user input into structured, typed objects.

> Flow:
> Raw Command String -> Command Dispatcher -> Argument Parser -> **MultiArgumentContext** -> Command Executor


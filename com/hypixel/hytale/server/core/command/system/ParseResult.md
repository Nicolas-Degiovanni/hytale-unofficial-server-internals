---
description: Architectural reference for ParseResult
---

# ParseResult

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Transient

## Definition
```java
// Signature
public class ParseResult {
```

## Architecture & Concepts
The ParseResult class is a state-holding object that encapsulates the outcome of a command parsing operation. It embodies the Result Object pattern, providing a robust alternative to using exceptions for control flow during command validation. Its primary role is to track whether a parsing attempt has succeeded or failed and to aggregate human-readable failure reasons.

This component is central to the server's command system. When a user input string is processed, various argument parsers and validators are invoked. Instead of throwing an exception upon encountering invalid input, these validators mutate a shared ParseResult instance, marking it as failed and adding specific error messages. This allows the system to collect all validation errors from a single command attempt rather than halting at the first one.

A notable feature is the `throwExceptionWhenFailed` constructor parameter. This provides a bridge to exception-based error handling, allowing the object to throw a GeneralCommandException immediately upon failure. This dual-mode capability offers flexibility, enabling its use in both modern, error-accumulating workflows and legacy systems that expect exceptions.

## Lifecycle & Ownership
- **Creation:** A new ParseResult is instantiated at the beginning of a command parsing sequence, typically within a command dispatcher or a top-level command node. It is created fresh for every single command execution attempt.
- **Scope:** The object's lifetime is extremely short, confined to the scope of a single method call that parses and executes a command. It does not persist between commands or across server ticks.
- **Destruction:** The ParseResult instance becomes eligible for garbage collection as soon as the command processing logic completes and the result has been communicated to the command sender.

## Internal State & Concurrency
- **State:** The internal state of ParseResult is **highly mutable**. It is initialized in a success state (`failed` is false). The `fail` method transitions it to a failed state, a one-way operation. The `reasons` list is a mutable collection that caches Message objects, growing as more failures are reported.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use within the context of a single command's execution. The internal list of reasons is not synchronized, and concurrent calls to `fail` would result in a race condition and potential data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fail(Message reason, Message... other) | void | O(N) | Marks the result as failed and appends one or more reason messages. Throws GeneralCommandException if configured to do so. N is the number of other messages. |
| failed() | boolean | O(1) | Returns true if the parsing operation has been marked as failed. |
| sendMessages(CommandSender sender) | void | O(M) | Sends all accumulated failure messages to the specified CommandSender. M is the number of stored messages. |

## Integration Patterns

### Standard Usage
The intended use is to create an instance, pass it through a chain of validation methods, and then check its final state to determine the next course of action.

```java
// A validator method receives the result object to mutate it on failure
public void validateUsername(String input, ParseResult result) {
    if (input.length() < 3) {
        result.fail(Message.raw("Username is too short."));
    }
}

// The command executor orchestrates the process
ParseResult result = new ParseResult();
validateUsername(someInput, result);
// ... other validations

if (result.failed()) {
    result.sendMessages(commandSender);
} else {
    // Proceed with command execution
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not reuse a ParseResult instance across multiple, independent command parsing operations. Its state is not reset and failure reasons will accumulate incorrectly.
- **Asynchronous Mutation:** Do not pass a ParseResult to an asynchronous task. The state is not synchronized and is only safe to modify on the thread that created it.

## Data Pipeline
ParseResult does not process data itself; rather, it is the container for metadata *about* a data processing pipeline. It tracks the validity of data as it flows through the command system.

> Flow:
> Raw Command String -> Command Dispatcher -> Argument Parser -> **ParseResult** (is mutated) -> Command Executor (is inspected) -> CommandSender (receives messages from **ParseResult**)


---
description: Architectural reference for CommandValidationResults
---

# CommandValidationResults

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Transient

## Definition
```java
// Signature
public class CommandValidationResults extends ValidationResults {
```

## Architecture & Concepts
The CommandValidationResults class is a specialized component within the server's command processing framework. It serves as a crucial bridge between the generic, engine-wide **Validation System** (inherited from ValidationResults) and the specific **Command Parsing System**.

Its primary architectural role is to act as a temporary, stateful container for validation failures that occur while parsing and validating the arguments of a user-submitted command. After the validation phase is complete, this object is responsible for translating the collected, low-level validation errors into a single, coherent failure state within a ParseResult object. This ensures that command validation is decoupled from command execution, allowing for a clean and robust parsing pipeline.

This class is not a long-lived service but rather a short-lived data object whose existence is scoped to a single command parsing operation.

### Lifecycle & Ownership
- **Creation:** An instance is created by the command parsing engine immediately before it begins validating the arguments for a specific command. It is initialized with ExtraInfo, providing contextual data from the codec layer.
- **Scope:** The object's lifetime is strictly confined to the validation and parsing phase of a single command. It does not persist beyond this operation.
- **Destruction:** It becomes eligible for garbage collection as soon as the `processResults` method has been invoked and the command parser moves on to the next stage (either execution or error reporting).

## Internal State & Concurrency
- **State:** CommandValidationResults is fundamentally **Mutable**. Its primary purpose is to accumulate a list of validation errors in its internal state, which is inherited from the parent ValidationResults class. This state is processed and subsequently cleared by the `processResults` method.

- **Thread Safety:** This class is **Not Thread-Safe** and must not be shared across threads. It is designed to be created, populated, and processed within the confines of a single thread that is handling a specific command request. Concurrent modification of its internal error list would lead to unpredictable behavior and race conditions.

## API Surface
The public contract is minimal, exposing only the functionality required by the command parsing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| processResults(ParseResult) | void | O(N) | Consumes all accumulated validation errors. If any represent a failure, it mutates the provided ParseResult to a failed state, populating it with a formatted error message. Clears internal state on success. |

## Integration Patterns

### Standard Usage
This class is intended to be used exclusively by the server's internal command handling logic. A command parser would use it to collect validation outcomes and finalize the parsing result.

```java
// Conceptual example within a command parser
ParseResult parseResult = new ParseResult();
CommandValidationResults validation = new CommandValidationResults(codecExtraInfo);

// ... argument validators are run, populating the 'validation' object ...
validator.validate(argument, validation);

// Finalize the parsing by processing any validation failures
validation.processResults(parseResult);

if (parseResult.isSuccess()) {
    // Proceed with command execution
} else {
    // Report failure to the command sender
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not attempt to reuse a CommandValidationResults instance across multiple command parsing operations. Its internal state is not designed for this and will contain stale data. A new instance must be created for each command.
- **Ignoring Processing:** Populating the object with validation results but failing to call `processResults` is a critical error. This will cause all validation failures to be silently ignored, potentially allowing invalid commands to execute.
- **External Modification:** Do not attempt to manually add or remove errors from its internal state from outside the designated validation framework. Rely on the `_processValidationResults` and `processResults` flow.

## Data Pipeline
This component sits at the end of the validation stage and acts as the gatekeeper to the command execution stage.

> Flow:
> Raw Command Arguments -> Codec Deserializers & Validators -> **CommandValidationResults** (Accumulator) -> `processResults` call -> Mutated ParseResult -> Command Dispatcher


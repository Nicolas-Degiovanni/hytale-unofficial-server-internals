---
description: Architectural reference for FlagArg
---

# FlagArg

**Package:** com.hypixel.hytale.server.core.command.system.arguments.system
**Type:** Transient

## Definition
```java
// Signature
public class FlagArg extends AbstractOptionalArg<FlagArg, Boolean> implements AbstractOptionalArg.DefaultValueArgument<Boolean> {
```

## Architecture & Concepts
The FlagArg class is a specialized component within the server's command system framework. It serves as a concrete, declarative model for a boolean command-line flag, such as *--force* or *--verbose*. Its primary architectural role is to encapsulate the name, description, and parsing behavior of an optional flag that does not take an explicit value; its presence in a command string implies a value of **true**.

This class is a specific implementation of the more generic AbstractOptionalArg, hard-wiring its behavior to the BooleanFlagArgumentType. This design decouples the definition of a command's structure from the logic of parsing. Command authors interact with this high-level construct, while the underlying command system uses the metadata it provides to correctly interpret and validate user input.

## Lifecycle & Ownership
-   **Creation:** A FlagArg instance is created declaratively by a developer during the definition phase of a new server command. It is constructed and registered with an AbstractCommand instance, establishing a direct ownership link.
-   **Scope:** The object's lifetime is bound to the AbstractCommand it is registered with. As commands are typically registered once at server initialization and persist for the entire server session, a FlagArg instance is a long-lived object.
-   **Destruction:** The object has no explicit destruction logic. It is eligible for garbage collection when the server shuts down and the central command registry, along with all registered commands, is cleared from memory.

## Internal State & Concurrency
-   **State:** The internal state of a FlagArg instance (name, description, argument type) is **effectively immutable** after construction. It acts as a read-only data structure or blueprint.
-   **Thread Safety:** This class is inherently **thread-safe**. As a stateless definition object, it can be safely read by multiple command execution threads simultaneously without locks or synchronization. The parsing process, which is stateful, operates on per-execution data and does not mutate the FlagArg instance itself.

## API Surface
The public API is minimal, focusing on providing metadata to the command system and facilitating fluent command construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FlagArg(command, name, desc) | Constructor | O(1) | Creates a new flag definition. |
| getDefaultValue() | Boolean | O(1) | Returns **false**. Defines the contract that a missing flag resolves to false. |
| getUsageMessage() | Message | O(1) | Generates a detailed, multi-line help message for this flag. |
| getUsageOneLiner() | Message | O(1) | Generates a concise, single-line usage string, e.g., [--force]. |

## Integration Patterns

### Standard Usage
A FlagArg is intended to be instantiated and registered as part of a command's definition, typically within a command builder or registration sequence.

```java
// Example of defining a command with a FlagArg
// NOTE: CommandBuilder is a hypothetical class for illustration.
CommandBuilder.create("teleport")
    .withArgument(new PlayerArg("target", "The player to teleport."))
    .withArgument(new LocationArg("destination", "The target location."))
    .withArgument(new FlagArg(
        this, // The AbstractCommand instance
        "silent",
        "If present, no teleport messages will be broadcast."
    ))
    .register();
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Usage:** Do not attempt to store or modify state on a FlagArg instance after its initial construction. It is a static definition, not a container for runtime data.
-   **Use Outside Command System:** This class is tightly coupled to the server's command parsing lifecycle. Instantiating it for any other purpose is not supported and will lead to unexpected behavior.

## Data Pipeline
The FlagArg class is a key component in the data transformation pipeline from raw user input to a structured, executable command context.

> Flow:
> Raw Command String (`/teleport Player1 10 20 30 --silent`) -> Command Parser -> **FlagArg Definition** (matched by name "silent") -> BooleanFlagArgumentType -> Parsed Value (`true`) -> Command Execution Context -> Command Logic Accesses `silent=true`


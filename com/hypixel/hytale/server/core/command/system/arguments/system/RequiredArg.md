---
description: Architectural reference for RequiredArg
---

# RequiredArg

**Package:** com.hypixel.hytale.server.core.command.system.arguments.system
**Type:** Transient

## Definition
```java
// Signature
public class RequiredArg<DataType> extends Argument<RequiredArg<DataType>, DataType> {
```

## Architecture & Concepts
The RequiredArg class is a fundamental component of the server's command system, representing a mandatory parameter that must be provided when a command is executed. It serves as a concrete implementation of the abstract Argument, enforcing a strict contract between the user's input and the command's signature.

Its primary role is not to parse data, but to define the metadata and constraints for a single argument: its name, description, and its non-optional nature. The actual parsing logic is delegated to a collaborator object, an ArgumentType, which is responsible for converting raw string input into the specified DataType (e.g., an Integer, String, or Player entity).

Within the command processing pipeline, the command parser iterates over a command's registered arguments. Upon encountering a RequiredArg, it asserts that sufficient input tokens are available. It then invokes the associated ArgumentType to consume and validate the input. A failure at this stage, either due to missing input or a parsing error, results in the immediate termination of command execution and the presentation of a usage error to the user.

This class is a key element in a **Composite** design pattern, where an AbstractCommand is composed of a collection of Argument objects (RequiredArg, OptionalArg, etc.) that collectively define its complete syntax.

### Lifecycle & Ownership
- **Creation:** An instance of RequiredArg is created exclusively during the definition phase of an AbstractCommand, typically within the command's constructor. It is instantiated by the developer defining the command's behavior and signature.
- **Scope:** The lifecycle of a RequiredArg instance is strictly bound to the AbstractCommand to which it is registered. It persists in memory as part of the command's definition for as long as the command itself is registered with the server's command registry.
- **Destruction:** The object is marked for garbage collection when its parent AbstractCommand is deregistered from the system, for instance, during a server shutdown or a module/plugin unload.

## Internal State & Concurrency
- **State:** **Effectively Immutable**. All internal fields, including the associated command, name, description, and ArgumentType, are set via the constructor and are not modified thereafter. This class is a data-carrier for the command's structural definition.
- **Thread Safety:** **Thread-Safe**. Due to its immutable nature, a RequiredArg instance can be safely read by multiple threads simultaneously without synchronization. This is critical, as command execution may be dispatched across different worker threads, all of which need to reference the command's argument definitions.

## API Surface
The public API is primarily focused on generating user-facing help and usage messages.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUsageMessageWithoutDescription() | Message | O(1) | Returns a compact usage string, typically for syntax summaries (e.g., `<name:type>`). |
| getUsageMessage() | Message | O(1) | Generates a detailed, multi-part help message including the name, type, and description. |
| getUsageOneLiner() | Message | O(1) | Generates the most common, concise usage string for command syntax errors (e.g., `<name>`). |

## Integration Patterns

### Standard Usage
A RequiredArg is instantiated and registered within the constructor of a class that extends AbstractCommand. This is the sole intended method for its creation and use.

```java
// Example: Defining a required "player" argument for a /kick command.
public class KickCommand extends AbstractCommand {
    public KickCommand() {
        super("kick", "Kicks a player from the server.");

        // 1. Define the required argument with its metadata and parsing type.
        RequiredArg<Player> target = new RequiredArg<>(
            this,
            "target",
            "The player to be kicked.",
            ArgumentTypes.PLAYER // A static provider for standard argument types
        );

        // 2. Register the argument with the command.
        this.addArgument(target);
    }

    @Override
    public void execute(CommandContext context) {
        // The parsed Player object is now available in the context.
        Player playerToKick = context.getArgument("target");
        // ... execution logic ...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Misuse for Optional Arguments:** Do not use RequiredArg for parameters that can be omitted by the user. This violates the class's contract and will cause the command parser to fail when the argument is not present. Use the `OptionalArg` class for such cases.
- **Using a Zero-Parameter Type:** The constructor explicitly forbids associating a RequiredArg with an ArgumentType that consumes zero input tokens (e.g., a simple flag). This is a logical contradiction, as a required argument must consume input. Attempting to do so will result in an `IllegalArgumentException`.
- **Stateful ArgumentType:** **WARNING:** The provided ArgumentType instance must be stateless and thread-safe. Passing an ArgumentType that contains mutable state can lead to severe concurrency bugs, as the same instance will be used across all executions of the command, potentially on different threads.

## Data Pipeline
The RequiredArg acts as a gatekeeper and metadata provider within the command parsing data flow. It does not transform data itself but enables the ArgumentType to do so.

> Flow:
> Raw Command String -> Command Parser -> **RequiredArg** (Validates presence of input) -> ArgumentType (Consumes and parses string token) -> Typed Data (e.g., Player instance) -> CommandContext -> Command Executor


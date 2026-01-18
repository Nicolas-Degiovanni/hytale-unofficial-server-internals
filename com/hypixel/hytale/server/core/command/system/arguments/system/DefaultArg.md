---
description: Architectural reference for DefaultArg
---

# DefaultArg

**Package:** com.hypixel.hytale.server.core.command.system.arguments.system
**Type:** Transient

## Definition
```java
// Signature
public class DefaultArg<DataType> extends AbstractOptionalArg<DefaultArg<DataType>, DataType> implements AbstractOptionalArg.DefaultValueArgument<DataType> {
```

## Architecture & Concepts
The DefaultArg class is a structural component within the server's Command System framework. It represents a command argument that is optional for the user to provide, but which resolves to a pre-configured, non-null default value if omitted during command execution.

Architecturally, it serves as a self-contained definition and data source. During the command *definition* phase, a developer instantiates a DefaultArg to declare an argument's name, type, description, and its fallback value. During the command *parsing* phase, the command system inspects the user's input. If a corresponding argument is not found, the parser queries the DefaultArg instance for its `getDefaultValue` to ensure the command's execution context is always fully populated.

This design decouples the command's execution logic from the complexities of parsing optional inputs, providing a robust and predictable contract for command handlers.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively during the definition of an AbstractCommand, typically within its constructor or an initialization block. The owning command instance is passed directly into the DefaultArg constructor, establishing a clear ownership link.
- **Scope:** The lifecycle of a DefaultArg instance is strictly bound to its parent AbstractCommand. It persists as long as the command is registered with the server's command registry.
- **Destruction:** The object is marked for garbage collection when its owning AbstractCommand is deregistered and no longer referenced. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state of a DefaultArg instance is **immutable**. All definitional properties, including the critical `defaultValue`, are declared as final and are set only once via the constructor. This object is a read-only data container after its creation.
- **Thread Safety:** This class is inherently **thread-safe**. Its immutable nature guarantees that multiple threads can access its methods, such as `getDefaultValue` or `getUsageMessage`, without any risk of race conditions or data corruption. No synchronization mechanisms are necessary. This is critical in a multi-threaded server environment where command definitions may be accessed concurrently.

## API Surface
The public API provides access to the argument's definition and its fallback value.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDefaultValue() | DataType | O(1) | Returns the configured default value. This is the primary contract for the command parsing system. |
| validateDefaultValue(ParseResult) | void | O(N) | Validates the *internally stored* default value against the rules of its ArgumentType. Typically called once at startup to prevent invalid command definitions. |
| getUsageMessage() | Message | O(1) | Generates a detailed, multi-part usage message for display in help commands. |
| getUsageOneLiner() | Message | O(1) | Generates a concise usage string suitable for command syntax summaries, e.g., [--name=value]. |
| getDefaultValueDescription() | String | O(1) | Returns the human-readable description of the default value's meaning. |

## Integration Patterns

### Standard Usage
A DefaultArg is defined as a field within an AbstractCommand subclass and initialized in its constructor. The command system later uses this definition to parse input and execute the command.

```java
// Inside a class extending AbstractCommand
public class MyCommand extends AbstractCommand {
    private final DefaultArg<Integer> teleportDelay;

    public MyCommand() {
        super("mycommand", "A sample command.");

        this.teleportDelay = new DefaultArg<>(
            this,
            "delay",
            "The delay in seconds before teleporting.",
            ArgumentTypes.INTEGER,
            5, // The default value
            "5 seconds" // Description of the default
        );

        // Register the argument
        this.addArgument(this.teleportDelay);
    }
    
    // ... command execution logic
}
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Instantiation:** Do not create DefaultArg instances during command parsing or execution. They are strictly for the declarative definition phase of a command's lifecycle.
- **Null Default Value:** The constructor and internal fields are annotated as Nonnull. Attempting to provide a null `defaultValue` will result in a runtime exception and is a violation of the class's contract.
- **State Mutation:** The class is immutable by design. Do not attempt to modify its state after construction using reflection or other unsupported mechanisms.

## Data Pipeline
The DefaultArg acts as a conditional data source within the command parsing pipeline. It injects its default value only when a user-provided value is absent.

> Flow:
> User Input String -> Command Parser -> Parser Fails to Find User-Provided Argument -> **Parser calls getDefaultValue() on DefaultArg** -> Default Value is added to Execution Context -> Command Handler Receives Populated Context


---
description: Architectural reference for OptionalArg
---

# OptionalArg

**Package:** com.hypixel.hytale.server.core.command.system.arguments.system
**Type:** Transient

## Definition
```java
// Signature
public class OptionalArg<DataType> extends AbstractOptionalArg<OptionalArg<DataType>, DataType> {
```

## Architecture & Concepts
The OptionalArg class is a foundational component of the server's command system, representing the schema for an optional, named argument that requires a value. It serves as a formal definition, or blueprint, that the command parser uses to validate and interpret user input.

This class exists within a hierarchy of argument types. It is distinct from a required argument, which must always be provided, and a Flag Argument, which is a valueless switch (e.g., --force). The core design principle is to separate the structural definition of an argument (its name, optionality, and description) from the logic of parsing its value. The parsing logic is delegated to a separate ArgumentType object, which OptionalArg holds by composition.

During command registration, instances of OptionalArg are attached to an AbstractCommand, forming a collection of argument definitions that collectively define the command's signature.

### Lifecycle & Ownership
- **Creation:** An OptionalArg is instantiated exclusively during the construction of an AbstractCommand subclass. The command developer creates and registers it as part of the command's definition. It is never created dynamically during command execution.
- **Scope:** The object's lifetime is strictly bound to its parent AbstractCommand. It persists in memory as part of the command's definition as long as that command is registered with the server's central command registry.
- **Destruction:** The object is marked for garbage collection when its parent command is unregistered from the server or during a complete server shutdown. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** The state of an OptionalArg instance is **effectively immutable**. All defining properties—the parent command, name, description, and associated ArgumentType—are provided via the constructor and cannot be changed during its lifetime. It is a pure data-holding object.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature, a single OptionalArg instance can be safely read by multiple threads simultaneously without locks or other synchronization primitives. This is critical, as the command system may process commands from multiple players or systems concurrently, all referencing the same static command definitions.

## API Surface
The public API is minimal, primarily focused on providing metadata for help generation and usage messages. The core logic is encapsulated within the command system's parsing loop, which interacts with the object's internal state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUsageMessage() | Message | O(1) | Generates a detailed, user-facing usage string for help menus, including the name, type, and description. |
| getUsageOneLiner() | Message | O(1) | Generates a compact, standardized usage string suitable for command syntax summaries, like [--name=?]. |

## Integration Patterns

### Standard Usage
The sole intended use of this class is within the constructor of a command definition to declare an optional argument. The command system then consumes this definition.

```java
// Inside the constructor of a custom command class
public class MyCommand extends AbstractCommand {
    public MyCommand() {
        super("mycommand", "A custom command.");

        // Define an optional argument named 'level' that expects an integer.
        this.addArgument(new OptionalArg<>(
            this,
            "level",
            "Sets the operational level.",
            ArgumentTypes.INTEGER
        ));
    }
    // ... command execution logic
}
```

### Anti-Patterns (Do NOT do this)
- **Instantiation without Registration:** Creating an instance of OptionalArg without passing it to the parent command's `addArgument` method (or equivalent) is pointless. The object will be created but immediately garbage collected, and the command system will be unaware of the argument.
- **Using Zero-Parameter Types:** The constructor explicitly throws an IllegalArgumentException if you provide an ArgumentType that consumes zero parameters. This is a design guardrail. For valueless switches, you **must** use a Flag Argument instead.

## Data Pipeline
OptionalArg acts as a schema during the command parsing pipeline. It does not process data itself but provides the rules for the parser to follow.

> Flow:
> Raw User Input String -> Command Parser -> **OptionalArg** (Schema validation) -> ArgumentType (Value parsing & type conversion) -> Parsed Command Context -> Command Executor


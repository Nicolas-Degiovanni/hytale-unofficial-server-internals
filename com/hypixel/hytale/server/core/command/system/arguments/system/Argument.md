---
description: Architectural reference for Argument
---

# Argument

**Package:** com.hypixel.hytale.server.core.command.system.arguments.system
**Type:** Transient

## Definition
```java
// Signature
public abstract class Argument<Arg extends Argument<Arg, DataType>, DataType> {
```

## Architecture & Concepts
The Argument class is the foundational abstract model for defining a single, parsable component of a server command. It acts as a schema or contract for a piece of user input, binding together a name, a data type, validation rules, and suggestion logic. It does not perform parsing itself; instead, it delegates the conversion of raw string input into a specific data type to an associated ArgumentType instance.

This class is a core element of the server's Command Definition framework. It employs the Curiously Recurring Template Pattern (CRTP) through its generic signature, `Arg extends Argument<Arg, DataType>`. This pattern enables a fluent, chainable API in concrete subclasses, allowing methods like `addValidator` to return the specific subclass type, enhancing developer ergonomics when constructing complex command structures.

An Argument is intrinsically linked to the AbstractCommand it is defined within. This tight coupling ensures that its lifecycle and configuration are managed within the context of a single, well-defined command.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively during the definition phase of an AbstractCommand. A developer creates concrete implementations of Argument (e.g., PlayerArgument, IntegerArgument) as part of a command's constructor or initialization block.
- **Scope:** The lifecycle of an Argument instance is identical to the AbstractCommand to which it is registered. It persists as long as the command definition is loaded by the server.
- **Destruction:** The object is eligible for garbage collection when its parent AbstractCommand is unregistered and unloaded from the server, typically during a plugin or world shutdown sequence.

## Internal State & Concurrency
- **State:** The class encapsulates the configuration for a command argument. Its state is mutable only during a specific pre-registration window. Key state includes the argument's name, its ArgumentType, an optional list of Validators, and an optional SuggestionProvider.

    **WARNING:** Once the parent AbstractCommand completes its registration with the server's command system, the Argument's state becomes effectively immutable. Any attempt to modify it by calling `addValidator` or `suggest` will result in an IllegalStateException. This is a critical design feature to ensure command definitions are stable and predictable at runtime.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be configured and used by a single thread—the server's main thread or a dedicated command processing thread. The "configure-then-use" lifecycle enforced by the registration check mitigates most concurrency risks, as the object is not modified after its initial setup. Concurrent access during the configuration phase is unsupported and will lead to undefined behavior.

## API Surface
The public API is designed for two distinct phases: configuration (building the command) and execution (using the parsed values).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addValidator(Validator) | Arg | O(1) | Adds a validation rule to be checked after parsing. Throws IllegalStateException if the command is already registered. |
| validate(DataType, ParseResult) | void | O(N) | Executes all registered validators against the parsed data. N is the number of validators. |
| get(CommandContext) | DataType | O(1) | Retrieves the parsed and validated value from the command context during execution. |
| suggest(SuggestionProvider) | Arg | O(1) | Attaches a custom suggestion provider for tab-completion. Throws IllegalStateException if the command is already registered. |
| getSuggestions(sender, text) | List<String> | O(N) | Gathers suggestions from both the attached SuggestionProvider and the underlying ArgumentType. |

## Integration Patterns

### Standard Usage
An Argument is defined as a field within an AbstractCommand subclass and configured in its constructor. During command execution, the parsed value is retrieved from the CommandContext.

```java
// In an AbstractCommand subclass definition
private final Argument<PlayerArgument, ServerPlayer> targetPlayer = 
    new PlayerArgument(this, "target")
        .addValidator(PlayerValidators.isOnline());

// In the command's execute method
@Override
public void execute(CommandContext context) {
    ServerPlayer player = context.get(this.targetPlayer);
    // ... logic to teleport the player
}
```

### Anti-Patterns (Do NOT do this)
- **Post-Registration Modification:** Never attempt to call `addValidator` or `suggest` on an Argument instance after the server has started or the command has been registered. The system is designed to fail-fast with an exception.
- **Instance Sharing:** Do not share a single Argument instance across multiple distinct AbstractCommand definitions. The internal reference `commandRegisteredTo` creates a tight, one-to-one coupling that would be violated.
- **Manual Validation Calls:** Do not call the `validate` method directly. The command parsing system is responsible for invoking validation at the correct point in its pipeline. Manual invocation bypasses critical parts of the parsing and error-reporting flow.

## Data Pipeline
The Argument class is a key component in the command processing pipeline. It acts as a schema and validation gatekeeper after the initial parsing is performed by its associated ArgumentType.

> Flow:
> Raw String Input -> Command Parser -> **ArgumentType** (Parses String to DataType) -> **Argument** (Validates DataType) -> CommandContext (Stores validated DataType) -> Command Executor (Retrieves DataType from Context)


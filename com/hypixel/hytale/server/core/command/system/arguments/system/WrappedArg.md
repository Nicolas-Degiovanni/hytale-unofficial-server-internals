---
description: Architectural reference for WrappedArg
---

# WrappedArg

**Package:** com.hypixel.hytale.server.core.command.system.arguments.system
**Type:** Design Pattern (Decorator/Wrapper)

## Definition
```java
// Signature
public abstract class WrappedArg<BasicType> {
```

## Architecture & Concepts
The WrappedArg class is a foundational component within the server's command system, implementing the **Decorator** design pattern. Its primary architectural role is to augment the functionality of a core Argument instance without altering the Argument's class structure. This promotes a flexible and composable system for defining command parameters.

This class acts as a transparent wrapper, forwarding most calls to the underlying Argument object it contains. Subclasses of WrappedArg introduce new behaviors, such as handling optionality, providing default values, or adding aliases. This approach decouples the fundamental parsing logic of an Argument (e.g., parsing an integer from a string) from higher-level command syntax features.

By using composition over inheritance, the system avoids a combinatorial explosion of classes. Instead of having classes like OptionalIntegerArgument, OptionalStringArgument, and so on, the framework combines a core IntegerArgument with an OptionalArgument wrapper (a subclass of WrappedArg).

## Lifecycle & Ownership
- **Creation:** Instances of WrappedArg are not meant to be instantiated directly. They are created implicitly by the command system's fluent API when a developer chains methods onto a base Argument definition. For example, calling a method like *optional()* on an argument builder will construct a specific subclass of WrappedArg.
- **Scope:** The lifetime of a WrappedArg instance is bound to the lifetime of the command definition it is part of. It is effectively a stateless configuration object that persists as long as its parent command is registered with the server.
- **Destruction:** The object is eligible for garbage collection when its parent command is unregistered or during server shutdown. It holds no persistent resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** WrappedArg is effectively **immutable**. Its only state is a final reference to the underlying Argument it decorates. All operations are either stateless or delegate state management to the wrapped object or the per-call CommandContext.
- **Thread Safety:** This class is inherently **thread-safe**. As an immutable wrapper, it can be safely accessed from multiple threads. However, the overall thread safety of a command execution depends on the implementation of the wrapped Argument and the CommandContext, which are typically confined to a single thread per command execution cycle.

## API Surface
The public contract of WrappedArg primarily involves delegation and conditional enhancement of the core Argument API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| provided(context) | boolean | O(1) | Delegates to the wrapped argument to check if a value was supplied in the command string. |
| getName() | String | O(1) | Delegates to the wrapped argument to retrieve its primary name. |
| getDescription() | String | O(1) | Delegates to the wrapped argument to retrieve its help text. |
| addAliases(aliases) | D | O(N) | Conditionally adds aliases. **Warning:** Throws UnsupportedOperationException if the wrapped argument is not an AbstractOptionalArg. |
| getArg() | Argument | O(1) | Exposes the underlying, undecorated Argument instance. |
| get(context) | BasicType | Varies | **Protected method.** Delegates to the wrapped argument to retrieve the parsed value from the context. Complexity depends on the wrapped argument's parsing logic. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with the WrappedArg class directly. Its use is an implementation detail of the command definition framework. The framework's fluent API is the intended point of interaction.

```java
// Example of a fluent API that would create a WrappedArg subclass internally
Command.newBuilder("teleport")
    .addArgument(
        // This .optional() call is what creates the wrapper
        Arguments.player("target").optional()
    )
    .build();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to subclass and instantiate WrappedArg manually. The command system relies on specific internal implementations created by its argument builders.
- **Misusing Aliases:** Do not call addAliases on an argument that is not explicitly defined as optional. The system is designed to enforce that only optional arguments can have aliases, and violating this will result in a runtime exception.

```java
// ANTI-PATTERN: This will throw an UnsupportedOperationException
Argument requiredArg = Arguments.integer("level");
WrappedArg wrapped = new SomeConcreteWrappedArg(requiredArg);

// This call is invalid because the wrapped argument is required, not optional.
wrapped.addAliases("lvl");
```

## Data Pipeline
WrappedArg functions as a node in the command processing pipeline, specifically during the argument parsing and value resolution phase.

> Flow:
> Raw Command String -> Command Dispatcher -> Argument Parser -> **WrappedArg (and its underlying Argument)** -> CommandContext -> Command Executor


---
description: Architectural reference for WrappedArgumentType
---

# WrappedArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Abstract Component

## Definition
```java
// Signature
public abstract class WrappedArgumentType<DataType> extends SingleArgumentType<DataType> {
```

## Architecture & Concepts
The WrappedArgumentType class is an abstract implementation of the **Decorator** design pattern, forming a foundational component of the server's command argument parsing system. It is designed to augment or modify the behavior of another ArgumentType without altering its core implementation.

This class acts as a transparent wrapper, or "middleware," around a concrete ArgumentType instance. Its primary architectural purpose is to enable compositional behavior. Developers can chain wrappers to create complex validation, transformation, or suggestion logic. For example, one could wrap an IntegerArgumentType with a custom RangeArgumentType to enforce that the parsed integer falls within a specific bound, all without modifying the original integer parsing logic.

By extending SingleArgumentType, it signals to the command system that it resolves to a single value of type DataType, maintaining a consistent contract with the parser.

## Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses are created during the command definition phase. They are constructed by the developer when building a command's argument structure and are passed to the command registry. They are *not* managed by a dependency injection container.
- **Scope:** The lifecycle of a WrappedArgumentType instance is tightly coupled to the command it belongs to. It persists in memory as part of the command's definition for as long as that command is registered with the server.
- **Destruction:** The object is eligible for garbage collection when the associated command is unregistered or when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The internal state is minimal and effectively immutable. The core state consists of the final field named wrappedArgumentType, which holds a reference to the decorated ArgumentType. This reference is set at construction and cannot be changed.
- **Thread Safety:** This class is inherently thread-safe. It holds no mutable state related to a specific parsing operation. All parsing state is managed within the MultiArgumentContext object passed as a method parameter during parsing. Therefore, a single instance of a WrappedArgumentType subclass can be safely used by the command system to parse arguments from multiple sources concurrently, assuming the wrapped ArgumentType is also thread-safe.

## API Surface
The public API is designed for extension by subclasses and integration with the command parsing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getExamples() | String[] | O(1) | Returns command examples. Delegates to the wrapped type unless overridden. |
| get(MultiArgumentContext) | DataType | O(1) | A convenience method to retrieve the parsed value from the context object. |

## Integration Patterns

### Standard Usage
This is an abstract class and cannot be instantiated directly. The standard pattern is to extend it to create a new, specialized argument type that modifies an existing one.

```java
// Example: A wrapper that makes an argument optional
public class OptionalArgumentType<T> extends WrappedArgumentType<T> {

    public OptionalArgumentType(ArgumentType<T> wrappedType) {
        // Pass the wrapped type up to the parent constructor
        super(new Message("optional." + wrappedType.getName()), wrappedType, "[" + wrappedType.getArgumentUsage() + "]");
    }

    @Override
    public T parse(MultiArgumentContext context) throws CommandException {
        // Custom logic: attempt to parse, but return null if it fails
        // instead of throwing an exception.
        try {
            return this.wrappedArgumentType.parse(context);
        } catch (CommandException e) {
            return null;
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to use `new WrappedArgumentType(...)`. The compiler will prevent this as the class is abstract.
- **Omitting Super Constructor:** Subclasses MUST call `super(...)` in their constructor to properly initialize the wrapped argument type. Failure to do so will result in a compilation error and an invalid object state.
- **Stateful Implementations:** Avoid adding mutable instance fields to subclasses. Parsing logic should remain stateless and rely only on the provided MultiArgumentContext to ensure thread safety and predictability.

## Data Pipeline
WrappedArgumentType sits in the middle of the command parsing data flow. It intercepts the parsing process for a specific argument, applies its logic, and then typically delegates to the underlying type it wraps.

> Flow:
> Raw Command String -> Command Parser -> **WrappedArgumentType Subclass** -> Inner ArgumentType -> Parsed Value written to MultiArgumentContext -> Command Executor


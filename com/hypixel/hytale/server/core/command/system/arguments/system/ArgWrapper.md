---
description: Architectural reference for ArgWrapper
---

# ArgWrapper

**Package:** com.hypixel.hytale.server.core.command.system.arguments.system
**Type:** Utility / Data Transfer Object

## Definition
```java
// Signature
public record ArgWrapper<W extends WrappedArg<BasicType>, BasicType>(
   @Nonnull ArgumentType<BasicType> argumentType, @Nonnull Function<Argument<?, BasicType>, W> wrappedArgProviderFunction
) {
```

## Architecture & Concepts
The ArgWrapper is an immutable data carrier that acts as a factory for specialized command arguments. It is a core component of the server's command system, enabling a flexible and extensible argument parsing pipeline.

Its primary architectural function is to decouple the generic argument parsing mechanism from the domain-specific logic of a particular argument type. It achieves this by pairing a base ArgumentType (e.g., a string, an integer) with a provider function. This function is responsible for "wrapping" a basic, parsed argument into a more complex, context-aware type, known as a WrappedArg.

This pattern is a form of **Type Decoration** or **Factory**. For example, a base String argument type can be wrapped to become a PlayerName argument. The wrapper's function would contain the logic to take the parsed string and perform a player lookup, returning a rich PlayerName object instead of a simple String. This promotes composition over inheritance and keeps the core parsing engine clean and agnostic of game-specific concepts.

## Lifecycle & Ownership
- **Creation:** Instances are created during the server's command registration phase. Typically, a developer defining a new command with custom argument types will instantiate an ArgWrapper to link their custom logic to a base parser.
- **Scope:** Application-scoped. An ArgWrapper instance persists for the entire lifecycle of the server, as it forms part of a command's static definition.
- **Destruction:** The object is eligible for garbage collection only when the server is shutting down or if the associated command is dynamically unregistered.

## Internal State & Concurrency
- **State:** **Immutable**. As a Java record, its fields are final by definition. An ArgWrapper instance cannot be modified after creation. It serves as a static configuration object.
- **Thread Safety:** **Thread-safe**. Due to its immutability, an ArgWrapper can be safely accessed and used by multiple command execution threads concurrently without any need for external synchronization.

**WARNING:** While the ArgWrapper itself is thread-safe, the provided Function is not guaranteed to be. Implementors must ensure that the logic within the wrappedArgProviderFunction is re-entrant and free of side effects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| wrapArg(argument) | W | O(F) | Executes the provider function to transform a base Argument into its specialized WrappedArg type. Complexity is dependent on the function (F). |

## Integration Patterns

### Standard Usage
The intended use is to register a custom argument behavior with the command system. A developer defines how a basic parsed type should be elevated into a richer, domain-specific type.

```java
// Example: Creating a wrapper for a "Player" argument type
// that is based on a simple String parser.
ArgWrapper<PlayerArgument, String> playerArgWrapper = new ArgWrapper<>(
    ArgumentTypes.STRING, // The base parser to use
    (baseArgument) -> new PlayerArgument(baseArgument) // The function to create the specialized type
);

// This wrapper would then be registered within a command's definition.
```

### Anti-Patterns (Do NOT do this)
- **Stateful Provider Functions:** The wrappedArgProviderFunction should be pure and stateless. Do not implement logic that relies on or modifies external mutable state, as this will break thread safety and lead to unpredictable behavior when commands are executed in parallel.
- **Complex Logic in Constructor:** Avoid performing heavy computations or I/O operations within the provider function. The function should be a lightweight factory; defer complex operations like database lookups to the validation or execution phase of the WrappedArg itself.

## Data Pipeline
The ArgWrapper is a critical transformation step in the command parsing data pipeline. It converts raw, parsed data into a semantically rich object that the command's execution logic can work with.

> Flow:
> Raw Command String -> Command Parser -> Base ArgumentType Parsing -> **ArgWrapper.wrapArg()** -> Specialized WrappedArg -> Command Executor


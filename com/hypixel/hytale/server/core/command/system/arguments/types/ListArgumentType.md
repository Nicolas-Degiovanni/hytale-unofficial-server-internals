---
description: Architectural reference for ListArgumentType
---

# ListArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient Component

## Definition
```java
// Signature
public class ListArgumentType<DataType> extends ArgumentType<List<DataType>> {
```

## Architecture & Concepts
The ListArgumentType is a composite argument parser that implements the **Decorator** design pattern. It does not parse input itself; instead, it wraps another concrete ArgumentType and repeatedly invokes it to consume the entire remaining input stream of a command.

Its primary role is to transform a parser for a single value (e.g., IntegerArgumentType, PlayerArgumentType) into a parser for a variable-length list of those values. This enables commands that can operate on an arbitrary number of entities, such as `/kick player1 player2 player3`.

Architecturally, it is a fundamental component for creating flexible and powerful server commands. Unlike most other argument types that consume a fixed number of string tokens, ListArgumentType is "greedy" and will attempt to parse tokens until the input is exhausted. Because of this behavior, it must always be the **final argument** in a command's signature.

## Lifecycle & Ownership
- **Creation:** Instantiated directly by a developer during the command definition and registration phase. It is constructed by providing an instance of the ArgumentType it is intended to wrap.
- **Scope:** The object's lifecycle is bound to the command it is part of. It persists as long as its parent command is registered in the server's CommandManager.
- **Destruction:** De-referenced and eligible for garbage collection when the command registry is cleared, typically during a server shutdown or a hot-reload of game modules.

## Internal State & Concurrency
- **State:** The configuration of a ListArgumentType instance is **immutable**. The wrapped argument type is stored in a final field, established at construction time. The class itself holds no state related to any specific parsing operation; all transient data is managed within the scope of the parse method.
- **Thread Safety:** This class is inherently **thread-safe**. The parse method is a pure function with respect to the object's state. It operates exclusively on its method arguments and local variables. Multiple threads can invoke parse on a shared instance without risk of data corruption, assuming the wrapped ArgumentType is also thread-safe.

## API Surface
The public contract is minimal, focusing entirely on the parsing operation inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(input, parseResult) | List<DataType> | O(N) | Iteratively consumes the input array, invoking the wrapped parser for each element. N is the number of elements to parse. Returns null on failure. |
| isListArgument() | boolean | O(1) | A flag method indicating this type consumes all remaining arguments. Always returns true. |

## Integration Patterns

### Standard Usage
ListArgumentType is used to define the final argument of a command that accepts multiple values of the same type. It wraps a more primitive argument type.

```java
// Command definition for /teleport <target1> [target2] [target3]...
CommandBuilder.create("teleport")
    .withArgument("targets", new ListArgumentType<>(new PlayerArgumentType()))
    .executes(context -> {
        List<Player> playersToTeleport = context.getArgument("targets");
        // ... teleport logic
    });
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Ordering:** Placing a ListArgumentType before other arguments in a command signature will result in a parsing error. The greedy nature of this type will consume all subsequent tokens, preventing other arguments from ever being parsed.
  > **WARNING:** A ListArgumentType must always be the last argument in a command definition.

- **Nested Lists:** Wrapping a ListArgumentType within another ListArgumentType is not supported and leads to undefined behavior. The system is not designed for multi-dimensional list parsing from a flat string input.

## Data Pipeline
ListArgumentType acts as a dispatcher within the command parsing pipeline. It orchestrates the flow of data from the raw token stream to the wrapped sub-parser.

> Flow:
> Raw Command String -> Command Tokenizer -> **ListArgumentType.parse()** -> (Loop: Slice Tokens -> Wrapped ArgumentType.parse()) -> Aggregated List -> Parsed Command Context


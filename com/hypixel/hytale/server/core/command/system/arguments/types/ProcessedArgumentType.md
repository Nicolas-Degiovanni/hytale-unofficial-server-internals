---
description: Architectural reference for ProcessedArgumentType
---

# ProcessedArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Abstract Base Class / Template

## Definition
```java
// Signature
public abstract class ProcessedArgumentType<InputType, OutputType> extends ArgumentType<OutputType> {
```

## Architecture & Concepts

The ProcessedArgumentType is an abstract class that forms a critical link in the server's command processing pipeline. It embodies the **Decorator** design pattern, enabling the extension and transformation of existing argument parsers without modifying their original code.

Its primary architectural role is to decouple raw syntactic parsing from higher-level semantic validation and transformation. For example, a base ArgumentType might parse a raw string from the command input, while a concrete implementation of ProcessedArgumentType wraps it to convert that string into a fully-realized game object, such as a Player or an Entity.

This class acts as a processing stage in a pipeline. It first invokes an inner, wrapped ArgumentType to parse an initial value (the InputType). If successful, it then applies its own transformation logic via the abstract **processInput** method to produce the final, desired value (the OutputType). This creates a composable and extensible system for defining complex command arguments.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of ProcessedArgumentType are instantiated during the server's command registration phase. They are typically constructed as part of a command definition, often via a command builder or factory. They are never created on-the-fly during command execution.
- **Scope:** An instance of a ProcessedArgumentType exists for the entire lifecycle of the command it is associated with. It is stateless and can be reused for every execution of that command.
- **Destruction:** The object is marked for garbage collection when its parent command is unregistered from the CommandManager or during a server shutdown event.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal reference to the wrapped **inputTypeArgumentType** is final and assigned at construction. The class itself maintains no mutable state between calls.
- **Thread Safety:** This class is inherently thread-safe. However, the thread safety of a concrete implementation is dependent on the implementation of the **processInput** method. As command execution is generally handled by a single worker thread per command, race conditions are not a typical concern.

**WARNING:** Implementations of **processInput** must not modify shared, mutable state without proper synchronization, although this is strongly discouraged by design.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String[] input, ParseResult result) | OutputType | O(N + M) | Executes the two-stage parsing pipeline. N is the complexity of the inner parser; M is the complexity of the processInput implementation. Returns null on failure. |
| processInput(InputType data) | OutputType | Varies | **Abstract.** The core transformation logic that subclasses must implement. Converts the successfully parsed InputType into the final OutputType. |
| getInputTypeArgumentType() | ArgumentType | O(1) | Returns the wrapped, inner argument parser. |

## Integration Patterns

### Standard Usage
The primary pattern is to create a concrete class that extends ProcessedArgumentType to bridge a raw data type with a complex game object.

```java
// Assume a simple StringArgumentType exists for parsing a single word.
ArgumentType<String> nameParser = new StringArgumentType("name", ...);

// Create a concrete class to convert a player name string into a Player object.
public class PlayerArgumentType extends ProcessedArgumentType<String, Player> {

    public PlayerArgumentType() {
        // The super constructor receives the name, usage message, and the inner parser.
        super("player", new Message("A valid online player."), nameParser);
    }

    @Override
    public Player processInput(String playerName) {
        // This is the core transformation logic.
        Player foundPlayer = Server.getWorld().getPlayerByName(playerName);

        if (foundPlayer == null) {
            // The framework will detect a null return and mark the parse as failed.
            return null;
        }
        return foundPlayer;
    }
}

// This new type can now be used to define a command argument.
commandBuilder.withArgument("target", new PlayerArgumentType());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to use `new ProcessedArgumentType(...)`. It is an abstract class and must be extended.
- **Blocking Operations:** Do not perform slow or blocking operations (e.g., database queries, network requests) within the **processInput** method. This method is on the critical path for command execution and must be highly performant. All lookups should be against cached, in-memory data.
- **Complex Nesting:** Avoid chaining multiple layers of ProcessedArgumentType. While possible, it creates a deep and complex call stack that is difficult to debug when a parse failure occurs. Prefer a single, clear processing step.

## Data Pipeline
The class operates as a distinct stage within the larger command parsing data flow.

> Flow:
> Raw Command String -> Command Dispatcher -> `String[]` arguments -> Inner `ArgumentType.parse()` -> `InputType` (e.g., String) -> **ProcessedArgumentType.processInput()** -> `OutputType` (e.g., Player) -> Command Executor


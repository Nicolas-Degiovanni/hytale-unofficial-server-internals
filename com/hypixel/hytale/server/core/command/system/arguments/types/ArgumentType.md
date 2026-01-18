---
description: Architectural reference for ArgumentType
---

# ArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Utility

## Definition
```java
// Signature
public abstract class ArgumentType<DataType> implements SuggestionProvider {
```

## Architecture & Concepts
The ArgumentType class is an abstract base class that forms the cornerstone of the server's command parsing system. It establishes a formal contract for converting raw string input from a command into a strongly-typed, usable data object. This class embodies the **Strategy Pattern**, where each concrete subclass (e.g., IntegerArgumentType, PlayerArgumentType) provides a specific algorithm for parsing one segment of a command.

Architecturally, ArgumentType instances act as stateless, reusable parsing engines. The central command processor delegates the responsibility of interpreting command arguments to the appropriate ArgumentType, decoupling the core command logic from the specifics of data validation and conversion. This design makes the command system highly extensible; new argument types can be introduced without modifying the core parsing machinery.

An ArgumentType is also responsible for providing metadata about the argument it represents, such as its name, usage instructions, and examples, which are used by help systems and user-facing documentation. Furthermore, through the SuggestionProvider interface, it powers the server's command auto-completion feature.

## Lifecycle & Ownership
-   **Creation:** Concrete implementations of ArgumentType are intended to be singletons. They are typically instantiated once during server bootstrap and registered with a central command system registry. They are not created on a per-command basis.
-   **Scope:** An ArgumentType instance is global and persists for the entire lifetime of the server. Its purpose is to define a *type* of parsing, not to hold the state of a specific parsed value.
-   **Destruction:** Instances are destroyed during server shutdown when the command system's central registries are cleared.

## Internal State & Concurrency
-   **State:** The state of an ArgumentType instance is **effectively immutable**. Fields like name, argumentUsage, and examples are initialized in the constructor and are not modified thereafter. The class is designed to be stateless with respect to individual command invocations.
-   **Thread Safety:** ArgumentType and its concrete implementations must be **thread-safe**. The core parsing and suggestion methods operate exclusively on the parameters passed to them, without modifying any internal instance fields. This immutability ensures that a single instance can be safely used to parse commands from multiple players or systems concurrently without locks or race conditions.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String[], ParseResult) | DataType | Varies | **(Abstract)** The core parsing method. Converts raw string tokens into the target DataType. Returns null on failure and populates the ParseResult with error details. |
| suggest(sender, text, num, result) | void | Varies | Populates the SuggestionResult with potential auto-completions for the current input. The base implementation is a no-op. |
| processedGet(sender, context, arg) | DataType | Varies | A higher-level data retrieval method. Throws UnsupportedOperationException by default. Subclasses can override for context-aware lookups (e.g., resolving "@p" to the nearest player). |
| getName() | Message | O(1) | Returns the translatable name of this argument type (e.g., "Integer", "Player"). |
| getArgumentUsage() | Message | O(1) | Returns the translatable usage string for this argument, often used in help messages (e.g., "<player>"). |
| getNumberOfParameters() | int | O(1) | Returns the number of string tokens this argument type consumes from the input. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with ArgumentType directly. Instead, they use a command builder or a static factory (e.g., an Arguments class) that provides pre-registered instances of concrete ArgumentType implementations.

```java
// A developer defines a command, requesting a pre-existing argument type.
// The system internally uses the registered IntegerArgumentType.parse()
// method when a player executes "/sethealth <player> 100".

CommandBuilder.create("sethealth")
    .withArgument("target", Arguments.player())
    .withArgument("amount", Arguments.integer())
    .executes(context -> {
        // ... logic
    });
```

### Extension (Creating a Custom Type)
To support a new type of data in commands, a developer must extend ArgumentType and implement the abstract methods.

```java
// Example of a custom ArgumentType for a "GameTeam" object
public class GameTeamArgumentType extends ArgumentType<GameTeam> {

    public GameTeamArgumentType() {
        super("team", "<team_name>", 1, "red", "blue");
    }

    @Override
    public GameTeam parse(@Nonnull String[] input, @Nonnull ParseResult result) {
        GameTeam team = TeamRegistry.findByName(input[0]);
        if (team == null) {
            result.setErrorMessage("No team found with that name.");
            return null;
        }
        return team;
    }

    @Override
    public void suggest(...) {
        // Add all registered team names to the suggestion result
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Storing State:** Do not add mutable instance fields to an ArgumentType subclass to store per-command state. This will break thread safety and lead to severe, unpredictable bugs in a concurrent environment.
-   **Direct Instantiation:** Do not create a new instance of an ArgumentType for every command registration. This is inefficient. They are designed to be shared singletons retrieved from a central registry.

## Data Pipeline
The ArgumentType is a critical processing stage in the server's command handling pipeline.

> Flow:
> Raw Command String -> Command Tokenizer -> Command Parser -> **ArgumentType.parse()** -> Typed Data Object -> CommandContext -> Command Executor Logic


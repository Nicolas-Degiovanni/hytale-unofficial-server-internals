---
description: Architectural reference for ArgTypes
---

# ArgTypes

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Utility

## Definition
```java
// Signature
public final class ArgTypes {
```

## Architecture & Concepts

The ArgTypes class is the central, static registry for all command argument parsers within the server's command system. It functions as the definitive "type system" for console and in-game commands, providing the logic to convert raw string inputs into strongly-typed Java objects. When a command is defined, its parameters are bound to one of the static constants provided by this class.

This system is built on a few core abstractions:

*   **SingleArgumentType:** The most fundamental parser. It is designed to consume a single token from the command input string and convert it into an object. Examples include BOOLEAN, INTEGER, and STRING.
*   **MultiArgumentType:** A composite parser that consumes multiple consecutive tokens to build a single, more complex object. For example, the VECTOR3I type consumes three separate integer tokens to construct a Vector3i object. This allows for intuitive command structures like `/teleport 100 64 -50` instead of `/teleport "100 64 -50"`.
*   **ProcessedArgumentType:** A powerful decorator pattern that chains parsers. It takes the output of one ArgumentType and applies a transformation function to produce a final object. For instance, BLOCK_ID uses the BLOCK_TYPE_KEY parser to get a string like *hytale:stone* and then processes it into its internal integer representation. This promotes reusability and separation of concerns.
*   **AssetArgumentType:** A specialized parser for resolving asset identifiers (e.g., items, blocks, sounds) by querying the server's core asset management system.

Critically, many argument types are not pure functions; they depend on global server state. Parsers like PLAYER_REF and WORLD must query the active **Universe** singleton to resolve names into live game objects. This makes ArgTypes a key integration point between the command system and the core game simulation.

## Lifecycle & Ownership

*   **Creation:** The ArgTypes class is non-instantiable. All fields are static and final, initialized by the JVM class loader when the class is first referenced during server bootstrap.
*   **Scope:** The argument parser instances exist for the entire lifetime of the server process. They are global, stateless, and shared across all command executions.
*   **Destruction:** The objects are reclaimed by the JVM only upon server shutdown. No manual memory management is required.

## Internal State & Concurrency

*   **State:** The ArgTypes class and the individual parser instances it holds are fundamentally stateless. They are immutable objects that encapsulate parsing *logic*, not runtime data. The `parse` methods operate exclusively on their inputs to produce an output, without modifying any internal fields.
*   **Thread Safety:** The parsers themselves are thread-safe. However, a significant number of them (e.g., PLAYER_REF, WORLD, PLAYER_UUID) perform read operations on shared, mutable game state via the global `Universe.get()` accessor. Therefore, the overall thread safety of a command execution depends entirely on the concurrency guarantees of the underlying game state components being queried.

    **Warning:** While the parsing logic is safe, executing commands that use world-aware argument types from asynchronous threads may lead to race conditions if the underlying World or Player data is not properly synchronized. The command system's execution model must guarantee safe access to this state.

## API Surface

The public contract of ArgTypes consists of its static final fields, which represent the available parsers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BOOLEAN | SingleArgumentType | O(1) | Parses "true" or "false" into a Boolean. |
| INTEGER | SingleArgumentType | O(1) | Parses a numerical string into an Integer. Fails on non-numeric input. |
| STRING | SingleArgumentType | O(1) | Parses a single token or a quoted string. |
| PLAYER_REF | SingleArgumentType | O(P) | Resolves a player username into a PlayerRef object. Involves a linear scan of online players (P). |
| WORLD | SingleArgumentType | O(1) | Resolves a world name into a World object. Involves a map lookup. |
| RELATIVE_POSITION | MultiArgumentType | O(1) | Parses three double coordinate tokens (e.g., 10.5 ~-2.0 50) into a RelativeDoublePosition. |
| BLOCK_PATTERN | ProcessedArgumentType | O(N) | Parses a list of weighted block entries (e.g., [80%stone, 20%dirt]) into a BlockPattern object. |
| BLOCK_MASK | ProcessedArgumentType | O(N) | Parses a list of block mask rules (e.g., [!water, >grass]) into a combined BlockMask object. |

## Integration Patterns

### Standard Usage

ArgTypes are used declaratively when building a command. The command framework uses these definitions to parse and validate user input before the command's execution logic is ever invoked.

```java
// Example of a command definition using ArgTypes
Command teleportCommand = CommandBuilder.create("teleport")
    .withPermission("hytale.command.teleport")
    .withArgument("target", ArgTypes.PLAYER_REF)
    .withArgument("destination", ArgTypes.RELATIVE_POSITION)
    .executes(context -> {
        PlayerRef target = context.get("target");
        RelativeDoublePosition dest = context.get("destination");
        
        // Execution logic uses the fully parsed and validated objects
        target.teleport(dest.resolve(context.getSenderLocation()));
    });
```

### Anti-Patterns (Do NOT do this)

*   **Manual Parsing:** Do not invoke the `parse` method on an ArgType directly. The command system's front controller is responsible for managing the input stream, handling failures via the ParseResult object, and providing necessary context. Bypassing it can lead to unhandled exceptions and inconsistent error messaging.
*   **Reflection-based Modification:** Do not attempt to modify the static final fields of ArgTypes at runtime. The entire command system relies on the stability and immutability of these predefined parsers.
*   **Assuming Pure Functions:** Do not assume a parser is a pure function of its string input. Types like PLAYER_REF depend on the current world state and will produce different results at different times.

## Data Pipeline

The ArgTypes class is a critical stage in the command processing pipeline, responsible for semantic analysis and type conversion.

> Flow:
> Raw User Input String -> Command Tokenizer -> **ArgTypes Parser Selection** -> **Instance Parsing & Validation** -> CommandContext Population -> Command Executor Logic


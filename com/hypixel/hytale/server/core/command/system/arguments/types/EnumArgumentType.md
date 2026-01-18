---
description: Architectural reference for EnumArgumentType
---

# EnumArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class EnumArgumentType<E extends Enum<E>> extends SingleArgumentType<E> {
```

## Architecture & Concepts
The EnumArgumentType class is a specialized argument parser within the server's command system framework. Its primary function is to act as a type adapter, translating raw string input from a command into a strongly-typed Java Enum constant. This component is fundamental for creating robust and user-friendly commands that rely on a fixed set of choices, such as game modes, difficulty levels, or entity types.

Architecturally, it decouples the command's business logic from the complexities of input parsing and validation. By handling this translation, it provides several key benefits:
1.  **Type Safety:** Command execution logic receives a guaranteed valid Enum constant, eliminating the need for manual string comparisons and error handling.
2.  **Centralized Validation:** The parsing logic is defined once and reused across all commands that need to accept an enum value.
3.  **Enhanced User Experience:** On parsing failure, it generates a precise error message that includes "did you mean" suggestions, calculated using a fuzzy string distance algorithm. This significantly improves command usability for players.

It is designed to be configured once during command registration and used repeatedly for parsing throughout the server's lifetime.

## Lifecycle & Ownership
-   **Creation:** An instance of EnumArgumentType is created by the developer during the server's command registration phase. It is constructed with the specific Enum class it is expected to parse.
-   **Scope:** The object's lifetime is bound to the command definition it is part of. It persists in memory as long as its parent command is registered with the command system, which is typically for the entire server session.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or if its associated command is dynamically unregistered from the system.

## Internal State & Concurrency
-   **State:** The internal state of EnumArgumentType is **effectively immutable**. Upon construction, it caches an array of the enum's constants and a list of their names. These collections are never modified post-initialization. This design choice is critical for performance and thread safety, as it avoids repeated reflection or computation during parsing.

-   **Thread Safety:** This class is **unconditionally thread-safe**. Due to its immutable internal state, the parse method can be invoked concurrently from multiple server threads without any risk of race conditions or data corruption. No external synchronization or locking mechanisms are required when using this class.

## API Surface
The public contract is minimal, focusing exclusively on construction and the core parsing operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EnumArgumentType(name, enumClass) | constructor | O(N) | Constructs the parser. Caches all constants from the provided Enum class. N is the number of constants in the enum. |
| parse(input, parseResult) | E | O(N) | Attempts to match the input string to a known enum constant, case-insensitively. Returns the matched constant or null on failure. Populates the ParseResult with a detailed error message if no match is found. |

## Integration Patterns

### Standard Usage
EnumArgumentType should be instantiated declaratively when building a command. It is provided as the type definition for a specific command argument.

```java
// Example: Defining a /gamemode <mode> command
// This is a conceptual representation of the command builder API.
CommandBuilder.new("gamemode")
    .withArgument(
        "mode",
        new EnumArgumentType<>("mode", GameMode.class)
    )
    .executes(context -> {
        // The context provides the already-parsed, type-safe enum.
        GameMode selectedMode = context.getArgument("mode", GameMode.class);
        player.setGameMode(selectedMode);
    });
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation During Execution:** Do not create a new EnumArgumentType inside the command's execution block. This is highly inefficient as it will repeatedly reflect and cache the enum constants on every command invocation. Define it once during registration.

-   **Manual Parsing:** Do not manually fetch the string argument and then attempt to parse it with Enum.valueOf. This bypasses the framework's case-insensitivity, error reporting, and suggestion features, leading to a poor user experience.

## Data Pipeline
The class operates as a transformation step within the server's command processing pipeline.

> Flow:
> Raw Command String -> Command Dispatcher -> **EnumArgumentType.parse()** -> Strongly-Typed Enum Object -> Command Execution Logic


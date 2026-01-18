---
description: Architectural reference for GameModeArgumentType
---

# GameModeArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class GameModeArgumentType extends SingleArgumentType<GameMode> {
```

## Architecture & Concepts
The GameModeArgumentType is a specialized parser within the server's command processing system. Its sole responsibility is to translate a raw string argument, provided by a user via a command, into a valid GameMode enum. This class acts as a type adapter, decoupling the command's execution logic from the complexities of user input validation, alias handling (e.g., "a" for "adventure"), and error reporting.

By encapsulating this logic, it provides three key benefits to the command system:
1.  **Validation:** It guarantees that any command executor receiving a GameMode object has a valid, non-null value.
2.  **User Experience:** In case of a typo or invalid input, it generates helpful, context-aware error messages with suggestions, powered by a fuzzy string matching algorithm.
3.  **Centralization:** All GameMode parsing logic is defined in one place, ensuring consistency across all commands that require this argument type.

It is a fundamental building block for creating robust and user-friendly server commands.

### Lifecycle & Ownership
-   **Creation:** An instance of GameModeArgumentType is typically created once during the server's command registration phase. A command definition (e.g., for `/gamemode`) will declare its argument types, leading to the instantiation of this class.
-   **Scope:** The object's lifetime is tied to the command it is registered with. It persists as long as the command is available on the server.
-   **Destruction:** The instance is eligible for garbage collection when the associated command is unregistered or when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** Instances of this class are stateless. The core logic relies on a static, shared `GAMEMODE_MAP`. This map is populated once in a static initializer block and is never modified thereafter, making it effectively immutable for the lifetime of the server.
-   **Thread Safety:** This class is fully thread-safe. The `parse` method is a pure function with respect to the instance, operating only on its inputs and the effectively immutable static map. It can be safely invoked by multiple threads concurrently, which is a common scenario in a server environment where multiple players may execute commands simultaneously.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(input, parseResult) | GameMode | O(1) | Attempts to parse the input string into a GameMode. On success, returns the enum. On failure, it populates the ParseResult with a detailed error message and returns null. The complexity is effectively constant as it depends on a small, fixed-size map. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. It is designed to be registered with the command system's argument builder when defining a new command. The framework then invokes the `parse` method internally during command processing.

```java
// Conceptual example of registering the argument type
CommandBuilder.new("gamemode")
    .withArgument("mode", new GameModeArgumentType())
    .executes(context -> {
        GameMode mode = context.getArgument("mode", GameMode.class);
        // ... execution logic
    });
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Do not call the `parse` method directly. It relies on the `ParseResult` object, which is managed by the command system's lifecycle. Bypassing the framework will break error reporting and command flow.
-   **Stateful Extension:** Do not extend this class to add instance-specific state. Its design relies on being stateless for thread safety and reusability.

## Data Pipeline
The flow of data through this component is linear and part of the broader command dispatch pipeline.

> Flow:
> Player Command String -> Server Command Dispatcher -> **GameModeArgumentType.parse()** -> Parsed GameMode Enum -> Command Executor Logic
>
> Failure Flow:
> Player Command String -> Server Command Dispatcher -> **GameModeArgumentType.parse()** -> `parseResult.fail()` -> Error Message -> Player
---


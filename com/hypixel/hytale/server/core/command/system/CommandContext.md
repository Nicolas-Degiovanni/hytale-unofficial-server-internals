---
description: Architectural reference for CommandContext
---

# CommandContext

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Transient

## Definition
```java
// Signature
public final class CommandContext {
```

## Architecture & Concepts
The CommandContext is a stateful container object that encapsulates all information related to a single, in-flight command execution. It serves as the primary data transfer object (DTO) between the command parsing system and the command's business logic.

Its core architectural purpose is to decouple command implementations from the complexities of parsing and sender management. A command handler receives a fully populated CommandContext and operates on its structured data, rather than processing raw input strings. This class acts as the "payload" for a command, holding the sender, the raw input, the specific command being executed, and a map of parsed, validated, and type-converted arguments.

It is instantiated by the core command dispatcher at the beginning of the command processing pipeline and is progressively populated during the argument parsing phase. Once fully constructed, it is passed to the target command's execution method.

### Lifecycle & Ownership
-   **Creation:** A new CommandContext is instantiated by the central command dispatching service for every single command that is executed. It is never pre-allocated or reused.
-   **Scope:** The object's lifetime is extremely short, scoped exclusively to the execution of one command. It exists only on the call stack of the command handler.
-   **Destruction:** The object is eligible for garbage collection as soon as the command's execution method returns. There is no manual cleanup required. **WARNING:** Storing a reference to a CommandContext beyond the scope of its execution is a memory leak and a design violation.

## Internal State & Concurrency
-   **State:** The CommandContext is highly **mutable** during its construction phase. The internal maps for argument values and raw input are populated by the parsing system via the package-private `appendArgumentData` method. After being passed to a command handler, it should be treated as conceptually immutable, though it is not enforced by the type system.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and used by a single thread within the scope of a single command execution. The internal state, which relies on non-synchronized collections like Object2ObjectOpenHashMap, is vulnerable to race conditions. **WARNING:** Never share a CommandContext instance across threads or access it from an asynchronous callback without explicit and robust synchronization, which is strongly discouraged.

## API Surface
The public API provides read-only access to the results of the command parsing and execution environment.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(Argument) | DataType | O(1) | Retrieves the parsed, type-converted, and validated value for a given argument. May populate and return a default value. |
| getInput(Argument) | String[] | O(1) | Retrieves the raw string tokens from the input that were used to parse a given argument. |
| provided(Argument) | boolean | O(1) | Returns true if the user provided a value for the specified argument. |
| sendMessage(Message) | void | O(1) | Sends a formatted message back to the source of the command (the CommandSender). |
| isPlayer() | boolean | O(1) | A convenience method to check if the CommandSender is a Player entity. |
| senderAs(Class) | T | O(1) | Performs a checked cast of the CommandSender to a specific type. Throws SenderTypeException on type mismatch. |
| sender() | CommandSender | O(1) | Returns the raw CommandSender object, which could be a Player, the server console, or another entity. |

## Integration Patterns

### Standard Usage
A command implementation receives the CommandContext as the sole parameter to its execution method. It uses the context to retrieve arguments and interact with the sender.

```java
// Inside an AbstractCommand's execute method
public void execute(CommandContext context) {
    // Retrieve the target player argument
    Ref<Player> target = context.get(TARGET_PLAYER_ARG);
    
    // Retrieve an optional amount, which has a default value
    int amount = context.get(AMOUNT_ARG);

    // Perform game logic with the arguments
    givePlayerItems(target, "diamond", amount);

    // Send a confirmation message back to the sender
    context.sendMessage(Message.translation("server.commands.give.success"));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create a CommandContext using its constructor. It is the responsibility of the core command system to build and provide this object. Manually creating one will result in a disconnected and non-functional object.
-   **Storing References:** Do not store a CommandContext in a field or pass it to another system for deferred processing. Its data is only guaranteed to be valid for the immediate, synchronous execution of the command.
-   **State Mutation:** Do not attempt to modify the internal state of the CommandContext from within a command's execution logic. It should be treated as a read-only object once received.

## Data Pipeline
The CommandContext is the central data structure that flows through the server-side command processing pipeline.

> Flow:
> Raw String Input -> Command Dispatcher -> **CommandContext (Creation)** -> Argument Parser -> **CommandContext (Population)** -> Command Executor -> Command Logic (Reads from Context) -> **CommandContext (Sends Response)**


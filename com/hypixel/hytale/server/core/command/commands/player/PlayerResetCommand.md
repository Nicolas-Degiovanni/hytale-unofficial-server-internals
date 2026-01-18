---
description: Architectural reference for PlayerResetCommand
---

# PlayerResetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Transient

## Definition
```java
// Signature
public class PlayerResetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerResetCommand class implements the server-side logic for the player reset command. It conforms to the Command Pattern, encapsulating a single, high-impact administrative action into a discrete, executable object.

Its primary architectural role is to serve as a bridge between the server's command input system and the core game state management layer, represented by the Universe singleton. By inheriting from AbstractTargetPlayerCommand, it delegates the complex and error-prone tasks of argument parsing, player lookups, and permission validation to a standardized base implementation. This leaves the class with a single, focused responsibility: executing the reset operation on a pre-validated player target.

This command is considered a destructive, high-privilege operation. It triggers a deep and wide-ranging state change for a player, effectively wiping their progress and resetting them to a default state as defined by the Universe.

### Lifecycle & Ownership
-   **Creation:** A single instance of PlayerResetCommand is instantiated by the command registration system during server bootstrap. It is then stored in a central command map, keyed by its name, "reset".
-   **Scope:** The singleton instance persists for the entire server session. The execution context, however, is transient; the arguments passed to the execute method are valid only for the duration of that single command invocation.
-   **Destruction:** The instance is dereferenced and garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** PlayerResetCommand is a stateless class. It contains no mutable fields and its behavior is determined entirely by the arguments passed to its execute method. This design ensures that a single instance can safely handle concurrent command invocations if the underlying system were ever to become multi-threaded.
-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the execution context is critical. The execute method **must** be invoked from the main server thread, as the call to Universe.resetPlayer performs non-atomic, thread-unsafe modifications to the core game state, including entity and world data.

**WARNING:** Invoking the execute method from any thread other than the primary server thread will result in world corruption, race conditions, and server instability. The command system guarantees this safety contract.

## API Surface
The public contract is defined by its constructor and the overridden execute method from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerResetCommand() | constructor | O(1) | Creates an instance, registering the command name "reset" and its description key. |
| execute(...) | protected void | Variable | Executes the reset logic. Complexity is dominated by the Universe.resetPlayer call, which involves I/O and significant state manipulation. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and managed by the server's command system. An administrator or a player with sufficient permissions would trigger its execution by typing a command into the game client.

*Example of user-facing command:*
`/reset Player123`

The server's command dispatcher parses this input, identifies the PlayerResetCommand handler, resolves the target "Player123" into a valid PlayerRef, and then calls the execute method with the fully populated CommandContext.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Invocation:** Do not manually create an instance and call execute. This bypasses the critical infrastructure provided by the command system, including permission checks, argument validation, and target player resolution.
    ```java
    // ANTI-PATTERN: Bypasses the entire command framework
    new PlayerResetCommand().execute(context, ...);
    ```
-   **Misuse as a Generic API:** This class is a command handler, not a general-purpose player management utility. For programmatic player resets, developers should directly use the underlying API that this command calls.
    ```java
    // CORRECT: Use the core API for internal systems
    Universe.get().resetPlayer(playerRef);

    // INCORRECT: Using a command class as an API
    CommandSystem.dispatch("reset " + player.getName());
    ```

## Data Pipeline
The flow of data for a player reset operation begins with user input and terminates with a change in the core world state. PlayerResetCommand is a key component in the middle of this pipeline, responsible for translating the command intent into a direct engine action.

> Flow:
> Client Command Input -> Server Network Layer -> Command Parser -> **PlayerResetCommand.execute()** -> Universe.resetPlayer() -> Entity & World State Modification -> Confirmation Message -> Network Layer -> Client UI


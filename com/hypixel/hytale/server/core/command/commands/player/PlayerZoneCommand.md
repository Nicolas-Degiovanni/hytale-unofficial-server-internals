---
description: Architectural reference for PlayerZoneCommand
---

# PlayerZoneCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Transient

## Definition
```java
// Signature
public class PlayerZoneCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerZoneCommand is a server-side command implementation within the Hytale command system framework. It is designed to retrieve and display the current geographical zone and biome of a target player.

Architecturally, this class is a concrete leaf in a Command Pattern. It extends `AbstractTargetPlayerCommand`, which provides the foundational logic for parsing a player target from the command's arguments. PlayerZoneCommand's sole responsibility is to implement the `execute` method, which contains the specific logic for querying the player's location data.

This class acts as a read-only interface to the game state. It interacts with the `Player` component and its associated `WorldMapTracker` to fetch data, formats it into user-facing `Message` objects, and sends it back to the command's issuer via the `CommandContext`. It does not modify any game state.

### Lifecycle & Ownership
- **Creation:** An instance of PlayerZoneCommand is created by the server's command registration system during server bootstrap. It is registered under the name "zone".
- **Scope:** The command object instance is a stateless singleton that persists for the entire server session. The parameters passed to its `execute` method, such as the `CommandContext`, are scoped to a single command invocation.
- **Destruction:** The object is destroyed and garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. The only field, `MESSAGE_NO_DATA`, is a `static final` constant, making it immutable. All variable data required for an operation is provided as arguments to the `execute` method.
- **Thread Safety:** The class instance itself is inherently thread-safe due to its stateless design. However, the `execute` method is expected to be called exclusively from the main server thread. The objects it interacts with, such as `World` and `EntityStore`, are not guaranteed to be thread-safe and must only be accessed within the server's single-threaded game loop to prevent race conditions and concurrent modification exceptions.

## API Surface
The public API is minimal and primarily consists of the constructor and the overridden `execute` method, which forms the contract with the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerZoneCommand() | constructor | O(1) | Constructs the command, setting its name and description key. |
| execute(...) | protected void | O(1) | Executes the command logic. Retrieves the target player's zone and biome from the WorldMapTracker and sends the result to the command source. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly by developers in code. It is invoked by the server's command handler when a user types the corresponding command into the chat console.

```
// User input in chat
/zone
// or to target another player
/zone Notch
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerZoneCommand()` in your code. The command system manages the lifecycle of command objects. Manually creating an instance circumvents registration and will not work.
- **Manual Invocation:** Avoid calling the `execute` method directly. This bypasses the command system's parsing, permission checks, and context setup. If you need to get a player's zone information programmatically, you should access the `Player` component and its `WorldMapTracker` directly.

## Data Pipeline
The data flow for this command is a simple request-response loop initiated by a user.

> Flow:
> User Input (`/zone`) -> Server Command Parser -> Command System -> **PlayerZoneCommand.execute()** -> WorldMapTracker Query -> Message Formatting -> CommandContext -> Network Layer -> Client Chat Display


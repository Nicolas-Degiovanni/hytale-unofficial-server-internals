---
description: Architectural reference for ChunkMaxSendRateCommand
---

# ChunkMaxSendRateCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkMaxSendRateCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ChunkMaxSendRateCommand class is a server-side component within the Command System framework. It serves as a direct control interface for administrators to tune network performance related to world data streaming. By extending AbstractTargetPlayerCommand, it inherits the logic for targeting a specific player entity, which is essential as chunk sending is a per-player operation.

Its primary architectural role is to act as a translator between a high-level text command and a low-level state change on a specific game component. It decouples the command input and parsing logic from the core game mechanics of the ChunkTracker, which manages the queue and transmission rate of world chunks to a connected client. This class encapsulates the specific action of modifying chunk send rates, making the command system extensible and modular.

## Lifecycle & Ownership
- **Creation:** A single instance of ChunkMaxSendRateCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is not created on a per-execution basis.
- **Scope:** The object is a long-lived singleton managed by the command registry. It persists for the entire duration of the server session.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding command execution. Its fields, such as secArg and tickArg, are final and define the command's argument structure. They do not store data between calls. The state that is modified by this command is external, residing within the ChunkTracker component of the target player entity.
- **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on the main server thread. The execute method directly mutates the state of a ChunkTracker component, which is not a thread-safe object and is owned by the server's primary game loop. Invoking this command from any other thread will lead to race conditions and world state corruption.

## API Surface
The public contract is fulfilled by overriding the execute method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Executes the command logic. Retrieves the target player's ChunkTracker and updates its rate-limiting properties based on parsed arguments. Sends feedback messages to the command source. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a privileged user through the server console or in-game chat. The command system handles parsing the input, resolving the target player, and populating the CommandContext.

```java
// This code is conceptual and represents the server's internal handling.
// A user would type: /chunk maxsendrate PlayerName sec:32 tick:2

// 1. Command system parses input and finds the ChunkMaxSendRateCommand instance.
// 2. It resolves "PlayerName" to a valid PlayerRef.
// 3. It creates a CommandContext and invokes the command.
CommandContext ctx = create_context_from_user_input();
registeredCommand.execute(ctx, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkMaxSendRateCommand()` in game logic. The command must be registered with the server's command system to function correctly.
- **Manual Execution:** Avoid calling the `execute` method directly. The command system is responsible for providing a correctly configured CommandContext and resolving all entity references. Bypassing the command system will result in unpredictable behavior and likely cause NullPointerExceptions.

## Data Pipeline
This class functions as a control point rather than a data processing node. Its flow is one of command and control, triggering a state change in a separate system.

> **Control Flow:**
> User Input (e.g., `/chunk maxsendrate ...`) -> Server Command Parser -> **ChunkMaxSendRateCommand.execute()** -> Entity Component System -> Mutate ChunkTracker State
>
> **Feedback Flow:**
> **ChunkMaxSendRateCommand.execute()** -> CommandContext.sendMessage() -> Network Message -> Client Chat UI


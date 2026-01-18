---
description: Architectural reference for PlayerEffectApplyCommand
---

# PlayerEffectApplyCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.effect
**Type:** Transient

## Definition
```java
// Signature
public class PlayerEffectApplyCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PlayerEffectApplyCommand class is a server-side command handler responsible for processing the in-game command to apply status effects to players. It serves as a direct bridge between the user input layer (the command system) and the core game logic layer (the Entity Component System).

Its primary function is to parse, validate, and execute requests to modify a player's state by adding an `EntityEffect`. This class encapsulates the logic for two distinct use cases, a common pattern in the Hytale command framework:
1.  **Self-Application:** The base class handles applying an effect to the player who executed the command.
2.  **Other-Application:** A nested private class, `PlayerEffectApplyOtherCommand`, is registered as a usage variant to handle applying an effect to a different player specified as an argument.

This design separates concerns cleanly, allowing each variant to have its own permission requirements and argument structure while being grouped under a single parent command, `/player effect`. The class relies heavily on the command system's argument type system (ArgTypes) to resolve string inputs into concrete engine objects like `EntityEffect` and `PlayerRef`.

## Lifecycle & Ownership
-   **Creation:** A single instance of PlayerEffectApplyCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is not created on a per-request basis.
-   **Scope:** The object's lifecycle is tied to the server session. It is registered and held in memory by the `CommandRegistry` from server start to server stop.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its member fields, such as `effectArg` and `durationArg`, are immutable definitions for command arguments, configured once in the constructor. They do not store state related to any specific command execution. All state is passed into the `execute` method via the `CommandContext`.
-   **Thread Safety:** This class is thread-safe due to its stateless nature and the engine's execution model. The `AbstractPlayerCommand` base ensures that the `execute` method is invoked on the correct world thread for the target player. The nested `PlayerEffectApplyOtherCommand` reinforces this by explicitly dispatching its core logic onto the target world's thread using `world.execute(...)`.

**WARNING:** Bypassing the command system and invoking `execute` methods from an arbitrary thread will break thread-safety guarantees and lead to severe data corruption or crashes.

## API Surface
The public API is minimal, intended for use only by the command dispatch system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerEffectApplyCommand() | Constructor | O(1) | Initializes the command, defines its arguments, and registers the "other player" usage variant. |
| execute(...) | protected void | Varies | Executes the command logic. Retrieves components and applies the specified effect to the target player. Complexity depends on entity component lookups. |

## Integration Patterns

### Standard Usage
A developer or user does not interact with this class directly via code. Interaction occurs by typing the command into the game client. The server's command processor is responsible for resolving the input and dispatching it to this handler.

The following example illustrates the conceptual invocation by the command system:

```java
// This code is conceptual and executed by the command system internals.
// Do NOT replicate this pattern.

// 1. Command is parsed and a context is built.
CommandContext context = commandParser.parse("/player effect apply hytale:speed 30.0");

// 2. The appropriate command handler is located.
PlayerEffectApplyCommand handler = commandRegistry.findHandlerFor("player.effect.apply");

// 3. The handler is executed with the necessary world state.
// The system provides the correct store, ref, playerRef, and world.
handler.execute(context, store, ref, playerRef, world);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlayerEffectApplyCommand()`. The command system manages the lifecycle of command handlers. Manual instantiation will result in a non-functional command that is not registered with the server.
-   **Manual Execution:** Never call the `execute` or `executeSync` methods directly. Doing so bypasses critical infrastructure, including permission checks, argument validation, and thread dispatching. This will lead to inconsistent state, `NullPointerException`s, and severe threading violations.
-   **State Storage:** Do not add mutable member variables to this class to store state between executions. Command handlers must be stateless and re-entrant.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a change to an entity's component state within the game world.

> Flow:
> Player Chat Input (`/player effect apply...`) -> Network Packet -> Server Command Parser -> **PlayerEffectApplyCommand** (Argument Resolution) -> World Thread Scheduler -> `EffectControllerComponent.addEffect` -> Entity State Mutation -> Network Sync to Client


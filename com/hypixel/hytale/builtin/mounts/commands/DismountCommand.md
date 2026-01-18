---
description: Architectural reference for DismountCommand
---

# DismountCommand

**Package:** com.hypixel.hytale.builtin.mounts.commands
**Type:** Transient Command Handler

## Definition
```java
// Signature
public class DismountCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DismountCommand class is a server-side implementation of the Command pattern, responsible for handling all logic related to forcing a player entity to dismount. It serves as a direct interface between player input (or administrator action) and the server's core Entity Component System (ECS).

Its primary architectural function is to translate a high-level command into a low-level state change within the game world. This is achieved by targeting a specific player entity and removing its **MountedComponent**. The removal of this component is the catalyst that signals other game systems—such as physics and animation—to detach the player from its mount and restore normal player control.

The class is composed of two distinct execution paths:
1.  **Self-Dismount:** Handled by the parent class, which inherits from AbstractPlayerCommand. This provides a simplified, secure context for a player to affect their own entity.
2.  **Targeted Dismount:** Implemented in the private nested class DismountOtherCommand. This variant handles the more complex case of an authorized user dismounting another player, requiring explicit entity lookups and careful, thread-safe world state modification.

## Lifecycle & Ownership
-   **Creation:** An instance of DismountCommand is created by the server's central command registry during the server bootstrap or plugin loading phase. It is discovered through reflection or explicit registration.
-   **Scope:** The command handler object is a long-lived service. A single instance persists for the entire lifecycle of the server session. It does not hold per-player state.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** DismountCommand is fundamentally stateless. All necessary data for execution (such as the target player, the world, and the ECS store) is provided as method parameters within the CommandContext at the time of invocation. The class holds only final, static references to translatable message keys.

-   **Thread Safety:** The class is designed for a multi-threaded server environment and strictly adheres to Hytale's concurrency model.
    -   The primary `execute` method, inherited from AbstractPlayerCommand, is guaranteed by the framework to be called on the appropriate world's main thread, making direct ECS modifications safe.
    -   The nested DismountOtherCommand demonstrates a critical concurrency pattern. Its `executeSync` method may be called from a generic command-handling thread. It performs initial validation and then explicitly schedules the state-mutating logic (component removal) onto the target world's main execution thread via `world.execute`.

    **WARNING:** Bypassing the `world.execute` scheduler for ECS modifications from an asynchronous context will lead to race conditions, data corruption, and server instability.

## API Surface
The primary contract is not a traditional method API but its registration as a command handler. The framework interacts with its protected `execute` methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | protected void | O(1) | Executes the dismount logic for the command-issuing player. Assumes execution on the correct world thread. |
| DismountOtherCommand.executeSync(context) | protected void | O(1) | Executes the dismount logic for a specified target player. Schedules the core logic to run on the target's world thread. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by players or administrators through the server's chat or console interface. The framework handles parsing, permission checks, and routing the request to this handler.

A server administrator dismounting another player:
```
// This is a conceptual representation of a console command.
// The server command parser resolves "PlayerName" to a PlayerRef
// and invokes the DismountOtherCommand variant.

/dismount PlayerName
```

A player dismounting themselves:
```
// The player issues the command with no arguments.
// The framework invokes the primary execute method from AbstractPlayerCommand.

/dismount
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new DismountCommand()`. The object has no utility outside of the command registry. It must be registered with the server's command system to function.

-   **Manual Invocation:** Do not call the `execute` or `executeSync` methods directly from other game systems. This bypasses the command framework's critical validation, permission handling, and context setup. If you need to dismount a player programmatically, you should use a dedicated API on the entity or a relevant game system, not by simulating a command.

-   **Unscheduled State Mutation:** Do not replicate the logic of DismountOtherCommand without scheduling the ECS modification on the world thread. Any call to `store.tryRemoveComponent` must be synchronized with the world's tick loop.

## Data Pipeline
DismountCommand acts as a controller in a command-and-control flow, not a data processing pipeline. It initiates a state change that propagates through the ECS and game logic.

> Flow:
> Player Chat Input (`/dismount <player>`) -> Server Network Layer -> Command Parser & Validator -> **DismountCommand.executeSync** -> World Task Scheduler -> **ECS State Change** (`store.tryRemoveComponent`) -> Game Systems (Physics, Animation) -> Network State Sync -> Client-Side Visual Update (Player is dismounted)


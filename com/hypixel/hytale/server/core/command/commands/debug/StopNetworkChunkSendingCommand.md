---
description: Architectural reference for StopNetworkChunkSendingCommand
---

# StopNetworkChunkSendingCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Transient

## Definition
```java
// Signature
public class StopNetworkChunkSendingCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The StopNetworkChunkSendingCommand is a server-side debug command that provides administrators with direct control over the world streaming mechanism for a specific player. It functions as a highly specialized controller that interfaces with the server's Entity Component System (ECS) to modify runtime behavior.

Its primary architectural role is to act as a user-facing entry point to manipulate a low-level server subsystem. By inheriting from AbstractPlayerCommand, it integrates into the server's command processing pipeline and is guaranteed to execute within the context of a specific player entity.

The command's sole function is to locate the target player's **ChunkTracker** component and toggle its *isReadyForChunks* state. This state is a critical flag read by the world streaming and networking systems. When set to false, the server will cease sending new world chunk data to that player's client, effectively isolating them from world updates. This is an invaluable tool for debugging client-side performance, network saturation, or entity behavior without the overhead of continuous world data transmission.

## Lifecycle & Ownership
-   **Creation:** An instance of StopNetworkChunkSendingCommand is created by the server's command registration system at runtime when a player executes the corresponding command string (e.g., "/networkChunkSending false"). It is not a persistent object.
-   **Scope:** The object's lifetime is exceptionally brief, existing only for the duration of a single command execution. It is created, its execute method is called, and it is then immediately eligible for garbage collection.
-   **Destruction:** The instance is destroyed by the Java Garbage Collector once the execute method returns. It holds no references that would extend its lifecycle beyond the command invocation.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. The RequiredArg field is a final, immutable definition of the command's expected input. All operations within the execute method are performed on state passed in as parameters (the CommandContext and ECS references), not on instance fields.
-   **Thread Safety:** The class instance itself is thread-safe due to its stateless and transient nature. However, the execution context is not. The execute method performs mutations on components within the ECS (specifically, the ChunkTracker). It is fundamentally unsafe to invoke this method from any thread other than the primary server thread responsible for the target player's world simulation. The command system's design ensures this condition is met.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Locates and modifies the ChunkTracker component for the executing player. Sends a confirmation message back to the player. |

## Integration Patterns

### Standard Usage
This command is not intended for programmatic use. It is designed to be invoked by a server administrator or developer via the in-game chat console.

```sh
# Disable chunk sending for your player
/networkChunkSending false

# Re-enable chunk sending for your player
/networkChunkSending true
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new StopNetworkChunkSendingCommand()`. The command system manages the lifecycle. Direct creation bypasses the necessary context injection and parsing.
-   **Programmatic Invocation:** Do not acquire an instance of this command to call its execute method from other game systems. If you need to control chunk sending programmatically, you must interact directly with the ChunkTracker component via the ECS API. This command is strictly a human-to-system interface.

## Data Pipeline
This component acts as a control-flow initiator rather than a data processor. The flow is triggered by user input and results in a state change within a core game system.

> Flow:
> Player Chat Input -> Server Command Parser -> **StopNetworkChunkSendingCommand.execute()** -> ECS Component Mutation (ChunkTracker) -> World Streaming System (reads new state on next tick)


---
description: Architectural reference for MemoriesCapacityCommand
---

# MemoriesCapacityCommand

**Package:** com.hypixel.hytale.builtin.adventure.memories.commands
**Type:** Transient

## Definition
```java
// Signature
public class MemoriesCapacityCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The MemoriesCapacityCommand is a server-side command processor that implements the Command Pattern. It serves as a direct administrative interface for manipulating a core gameplay feature: the Player Memories system.

Its primary architectural role is to act as a bridge between user input (a text command from a player or console) and the server's underlying Entity Component System (ECS). The command translates a simple instruction into a direct, stateful mutation of a player's entity data. Specifically, it targets the **PlayerMemories** component, adjusting its capacity or removing it entirely.

This class is a terminal node in the command processing chain. It does not delegate work to other services; instead, it directly interacts with low-level systems like the ECS **Store** and the player's network **PacketHandler** to enact changes and notify the client.

### Lifecycle & Ownership
- **Creation:** A single instance of MemoriesCapacityCommand is instantiated by the server's **CommandSystem** during the bootstrap phase. The system discovers and registers all command classes at startup.
- **Scope:** The command object is a singleton that persists for the entire lifetime of the server. However, its execution is transient; the *execute* method is invoked with a new, short-lived context for each command invocation.
- **Destruction:** The object is de-referenced and eligible for garbage collection only when the server is shutting down or the module containing it is unloaded.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only instance field, capacityArg, is a final object that defines the command's argument structure. All state that is read or modified by the *execute* method belongs to external systems passed in as parameters, such as the ECS Store and the PlayerMemories component.

- **Thread Safety:** The object instance is inherently thread-safe due to its stateless design. However, the execution context is not. The *execute* method is designed to be called exclusively from the main server thread (or a dedicated world thread).

    **WARNING:** Manually invoking *execute* from an external thread will bypass engine-level thread safety guarantees for the ECS and will lead to race conditions, data corruption, and server instability. The CommandSystem's dispatcher ensures serialized, thread-safe execution.

## API Surface
The public contract is defined by its role as a command within the server framework. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command logic. Modifies the target player's **PlayerMemories** component and sends a corresponding status packet to the client. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is automatically discovered and invoked by the server's command processing system. A user with appropriate permissions would trigger its execution by typing the command in-game or via the server console.

Example of user invocation:
`/memories capacity SomePlayer 10`

This input is parsed by the CommandSystem, which then identifies the target player, resolves the arguments, and calls the *execute* method on the registered MemoriesCapacityCommand instance with the appropriate context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new MemoriesCapacityCommand()`. The command will not be registered with the server and will have no effect. The framework handles instantiation.
- **Manual Invocation:** Never call the *execute* method directly. This bypasses critical framework infrastructure, including permission checks, argument validation, and thread safety management. Doing so can leave the game state inconsistent or crash the server.

## Data Pipeline
The MemoriesCapacityCommand sits at the intersection of the command system, the game state (ECS), and the network layer. The flow of data and control is unidirectional.

> Flow:
> User Command Input -> Server Command Parser -> **MemoriesCapacityCommand.execute()** -> ECS Store Mutation (PlayerMemories) -> PlayerRef PacketHandler -> Network Packet (UpdateMemoriesFeatureStatus) -> Client UI Update


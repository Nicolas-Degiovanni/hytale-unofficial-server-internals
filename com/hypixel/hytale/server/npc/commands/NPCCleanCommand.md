---
description: Architectural reference for NPCCleanCommand
---

# NPCCleanCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCCleanCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The NPCCleanCommand is a concrete implementation of the Command Pattern, designed to integrate with the server's command processing system. It provides a high-performance, administrative function to remove all Non-Player Character (NPC) entities from a given world instance.

Architecturally, this class acts as a lightweight trigger. It does not contain any complex logic itself. Instead, it delegates the core removal operation to the server's underlying Entity Component System (ECS) framework. By extending AbstractWorldCommand, it signals to the command system that its execution is dependent on a fully loaded world context, preventing it from being run in an invalid state.

Its primary mechanism involves querying the world's EntityStore for all entities possessing an NPCEntity component and queuing their removal via a CommandBuffer. This design ensures that the command's execution is both efficient and safe within the engine's concurrent processing model.

### Lifecycle & Ownership
- **Creation:** A single instance of NPCCleanCommand is typically instantiated by the server's command registration service during the bootstrap phase. It is then registered under the name "clean" within the "npc" command group.
- **Scope:** The command object is stateless and persists for the entire server session. Its `execute` method is invoked each time a user runs the command, but the object itself does not change.
- **Destruction:** The instance is eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** NPCCleanCommand is **stateless and immutable**. It holds no internal state and its behavior is determined solely by the context provided during the `execute` call.

- **Thread Safety:** The class instance is inherently thread-safe due to its stateless nature. The `execute` method initiates a parallel operation, `forEachEntityParallel`, which is a critical performance feature. This command does not manage its own threads or locks; it relies entirely on the sophisticated concurrency guarantees of the underlying ECS framework. The use of a CommandBuffer is a key concurrency pattern, deferring all entity mutations to a designated synchronization point in the game loop, thus preventing race conditions and modification errors during the parallel enumeration.

## API Surface
The public contract is defined by the parent AbstractWorldCommand class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N/c) | Initiates a parallel, deferred removal of all NPCEntity instances. N is the number of NPC entities and c is the number of available processor cores. The operation is not immediate. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the server's command system in response to player or console input.

The conceptual invocation by the system looks like this:
```java
// System-level code (conceptual)
// A player types "/npc clean"
String commandName = "clean";
CommandContext context = CommandContext.fromPlayer(player);

// The system finds the registered NPCCleanCommand instance
AbstractWorldCommand command = commandRegistry.find("npc", commandName);

// The system invokes the command with the correct world context
if (command != null) {
   command.process(context); // process() internally calls execute()
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCCleanCommand()`. The command will not be registered with the server and will have no effect.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the command system's essential pre-processing, including permission checks, context validation, and argument parsing, which can lead to an unstable or insecure server state.

## Data Pipeline
NPCCleanCommand acts as a control-flow trigger rather than a data-processing unit. Its execution initiates a state change within the world's ECS.

> Flow:
> Player Input (`/npc clean`) -> Command Parser -> Command Dispatcher -> **NPCCleanCommand.execute()** -> EntityStore.forEachEntityParallel() -> CommandBuffer.removeEntity() -> ECS Synchronization Point -> World State Updated


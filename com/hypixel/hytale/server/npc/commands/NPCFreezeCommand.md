---
description: Architectural reference for NPCFreezeCommand
---

# NPCFreezeCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCFreezeCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The NPCFreezeCommand is a state-mutating command that operates within the server's Entity Component System (ECS) architecture. It serves as the primary user-facing entry point for altering the motile state of Non-Player Character (NPC) entities.

Its core responsibility is to attach or detach the **Frozen** component to one or more entities. The Frozen component is a "tag component"—it contains no data and its mere presence on an entity signals a state change. Downstream systems, such as AI behavior trees, physics processors, and animation controllers, are responsible for querying for the Frozen component and altering their own execution logic accordingly.

This command, therefore, does not implement the freezing logic itself; it acts as a high-level orchestrator that triggers a state change in the world. This decouples the command input system from the underlying game mechanics, allowing for greater modularity.

The command supports multiple modes of operation:
*   **Single Target:** Freezes or thaws a specific NPC.
*   **Toggle:** Switches the frozen state of a specific NPC.
*   **Global:** Applies the Frozen component to all NPC and Item entities in the world.

## Lifecycle & Ownership
- **Creation:** A single prototype instance of NPCFreezeCommand is instantiated by the central CommandSystem during server bootstrap. It is discovered via reflection and registered under the name "freeze" within the "npc" command group.
- **Scope:** The command definition object persists for the entire server session as a singleton. However, each invocation of the command is a distinct, transient operation. The state for a given execution is contained entirely within the CommandContext object passed to the execute method.
- **Destruction:** The object is dereferenced and garbage collected only when the server shuts down and the CommandSystem registry is cleared.

## Internal State & Concurrency
- **State:** The NPCFreezeCommand object is stateless and effectively immutable after construction. Its fields are final references to argument parsers (FlagArg, EntityWrappedArg) which define the command's syntax. It holds no data related to any specific execution.
- **Thread Safety:** This class is thread-safe. The execute method is invoked by the main server thread. For the global `--all` operation, it leverages the `forEachEntityParallel` method, which distributes work across multiple threads.

    **WARNING:** To prevent race conditions and ensure data integrity during parallel iteration, component modifications are not applied directly. Instead, they are queued into a thread-local `commandBuffer`. These buffered commands are then flushed and applied sequentially by the ECS framework after the parallel processing is complete. This is a critical pattern for safe, high-performance, concurrent world modification.

## API Surface
The public contract is fulfilled by overriding the `execute` method from its parent, AbstractWorldCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N) or O(log N) | Executes the freeze/thaw logic. Complexity is O(N) when using the --all flag, where N is the total number of entities. For a single target, complexity is O(log N) or better, dependent on the underlying EntityStore lookup implementation. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is automatically discovered and invoked by the server's CommandSystem in response to player or console input. The framework is responsible for parsing the input, resolving arguments, and passing the required context to the `execute` method.

A user would invoke this command via the in-game chat or server console:
```sh
# Toggles the frozen state of the targeted NPC
/npc freeze --toggle

# Freezes all NPCs and Items in the world
/npc freeze --all
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCFreezeCommand()`. The instance created by the CommandSystem is the single source of truth. Creating a new instance will result in a disconnected object that is not registered to handle any commands.
- **Manual Execution:** Avoid calling the `execute` method directly. Bypassing the CommandSystem means skipping critical steps like permission validation, argument parsing, and context population. All command execution must flow through the central dispatcher.

## Data Pipeline
The data flow for this command begins with user input and ends with a change in the world state, which is then observed by other game systems.

> Flow:
> Player Chat Input (`/npc freeze ...`) -> Network Packet -> Server Command Parser -> **NPCFreezeCommand.execute()** -> ECS Command Buffer -> EntityStore State Change -> AI & Physics Systems (Observe `Frozen` component) -> Behavior Change (e.g., NPC stops moving)


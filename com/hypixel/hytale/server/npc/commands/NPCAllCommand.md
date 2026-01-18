---
description: Architectural reference for NPCAllCommand
---

# NPCAllCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCAllCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The NPCAllCommand class encapsulates a server-side administrative action triggered by a player. As a subclass of AbstractPlayerCommand, it integrates directly into the server's command processing system, making its functionality available via chat or console input.

Its primary architectural role is to serve as a high-level orchestrator for a complex bulk operation: spawning an instance of every available NPC role definition. It acts as a client to several other core server systems:

-   **NPCPlugin:** It queries this service to retrieve the master list of all registered NPC role templates. It then uses the plugin's factory methods to perform the actual entity spawning.
-   **Command System:** It leverages the argument parsing framework to accept an optional distance parameter, with built-in validation.
-   **Entity Component System (ECS):** After an NPC is spawned, this command directly manipulates the world's EntityStore to attach additional components, such as Nameplate and Frozen, to modify the final state of the newly created entities.
-   **World & Physics Utilities:** It uses NPCPhysicsMath to query the world geometry, ensuring NPCs are spawned at the correct height above the ground to prevent them from falling through the terrain.

This command is a classic example of the Command Pattern, isolating a specific, user-triggered server function from the core game loop and other systems.

### Lifecycle & Ownership
-   **Creation:** An instance of NPCAllCommand is created once by the server's command registration system during server startup or plugin loading. It is not instantiated per-execution.
-   **Scope:** The single instance is registered and persists for the entire lifetime of the server. Its internal state is constant after construction.
-   **Destruction:** The object is garbage collected when the server shuts down or when the NPCPlugin, which registers this command, is unloaded.

## Internal State & Concurrency
-   **State:** The NPCAllCommand object is effectively stateless between executions. It holds a final reference to its argument definition (distanceArg), but this is a configuration constant, not mutable state. All state required for execution, such as the target player, world reference, and entity store, is passed into the execute method.
-   **Thread Safety:** This class is **not thread-safe**. The execute method performs direct, unsynchronized modifications to the World and EntityStore. The server's architecture guarantees that all command executions are invoked serially on the main server thread. Invoking this method from any other thread will result in world state corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its inheritance from AbstractPlayerCommand. The primary entry point is the execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the command logic. Spawns N NPCs, where N is the total number of registered NPC roles. Complexity is linear based on the number of roles. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is designed to be discovered and executed exclusively by the server's command handling system in response to player input.

A player triggers this command by typing the following into the game's chat console:
`/npc all`
or with an optional distance:
`/npc all 10.5`

The server's internal dispatcher then invokes the command's execute method with the fully populated context.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new NPCAllCommand()`. The command system handles instantiation and registration. Manually creating an instance will result in a non-functional object that is not registered to respond to player input.
-   **External Invocation:** Do not call the execute method from other plugins or game systems. This bypasses the command system's permissions and context-building logic and, more critically, violates the server's threading model, which is a severe error.
-   **State Assumption:** Do not assume the command can be executed if the NPCPlugin is not fully initialized. The command relies on the plugin being in a valid state to retrieve role templates.

## Data Pipeline
The execution of this command initiates a one-way data flow that results in the creation of new entities in the world.

> Flow:
> Player Chat Input -> Server Command Parser -> **NPCAllCommand.execute()** -> NPCPlugin (Get Roles) -> Loop (For each role) -> NPCPhysicsMath (Get Height) -> NPCPlugin (Spawn Entity) -> EntityStore (Add Components) -> World State Update

---


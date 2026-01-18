---
description: Architectural reference for AmbienceEmitterAddCommand
---

# AmbienceEmitterAddCommand

**Package:** com.hypixel.hytale.builtin.ambience.commands
**Type:** Transient

## Definition
```java
// Signature
public class AmbienceEmitterAddCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The **AmbienceEmitterAddCommand** class is a server-side command handler that acts as a direct interface between a privileged user and the world's Entity Component System (ECS). It follows the Command Pattern, encapsulating all information needed to perform a single, discrete action: creating a new ambient sound emitter entity.

This class is not a persistent service but a transactional processor. Its primary architectural role is to translate a player's text-based command into a concrete change in the game world. It orchestrates the assembly of a new entity by creating and configuring a set of components—**AmbientEmitterComponent**, **TransformComponent**, **ModelComponent**, and others—and then committing this new entity to the **EntityStore**.

It is a critical tool for in-game level design and world building, allowing creators to dynamically place persistent sound sources without requiring external tools or server restarts.

### Lifecycle & Ownership
- **Creation:** An instance of **AmbienceEmitterAddCommand** is created and registered by the server's command system during the bootstrap phase, typically when the **AmbiencePlugin** is loaded. It is not instantiated on a per-request basis.
- **Scope:** The command object persists for the entire server session, held by the central command registry. However, the scope of its execution, via the **execute** method, is atomic and transient, lasting only for the duration of a single command invocation.
- **Destruction:** The object is de-referenced and eligible for garbage collection when the server shuts down or the parent plugin is unloaded.

## Internal State & Concurrency
- **State:** The class maintains an immutable internal state configured at construction time, specifically the definition of its required command arguments (**soundEventArg**). The **execute** method is designed to be re-entrant but not concurrent; it contains no instance fields that are mutated during execution. All state related to a specific invocation is confined to local variables within the method's stack frame.

- **Thread Safety:** This class is **not thread-safe** and is designed to be invoked exclusively by the server's main game thread. The **execute** method directly manipulates the world state by adding an entity to the **EntityStore**. Calling this method from any other thread will lead to world state corruption, race conditions, and server instability. The command system framework is responsible for ensuring single-threaded execution.

## API Surface
The public contract is defined by its inheritance from **AbstractPlayerCommand**. The primary entry point is the **execute** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Creates and spawns a new ambient emitter entity at the player's location. Throws **GeneralCommandException** if the executor is not a player. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the server's command dispatcher in response to player input. The standard interaction is through the in-game chat or console.

A programmatic invocation, if ever necessary, must be routed through the command system to ensure proper context and permissions are established.

```java
// Conceptual example of programmatic command dispatch
// WARNING: For internal engine use only.
CommandSystem commandSystem = server.getCommandSystem();
CommandSource source = getPlayerCommandSource(player);
String commandLine = "ambience emitter add hytale:music.overworld_ambience";

commandSystem.dispatch(source, commandLine);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call **new AmbienceEmitterAddCommand()**. The object is useless outside the context of the command registry, which handles its lifecycle and argument definitions.
- **Manual Execution:** Never call the **execute** method directly. Doing so bypasses critical framework logic, including argument parsing, permission checks, and thread safety guarantees. This is a high-risk action that can destabilize the server.
- **Stateful Modification:** Do not attempt to modify the command's internal state (e.g., its argument definitions) at runtime via reflection. This state is assumed to be immutable after construction.

## Data Pipeline
The command serves as a key component in the data flow from player input to world state modification. It transforms a high-level user intention into a low-level ECS operation.

> Flow:
> Player Chat Input -> Network Layer -> Server Command Parser -> **AmbienceEmitterAddCommand** -> EntityStore -> World State Update -> Network Replication to Clients


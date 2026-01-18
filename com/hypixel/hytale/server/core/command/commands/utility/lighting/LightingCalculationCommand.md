---
description: Architectural reference for LightingCalculationCommand
---

# LightingCalculationCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.lighting
**Type:** Singleton Command Handler

## Definition
```java
// Signature
public class LightingCalculationCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The LightingCalculationCommand class implements the Command Pattern to provide a user-facing interface for manipulating the server's core lighting engine. It acts as a specialized bridge between the server's command processing system and the ChunkLightingManager, which governs how light is calculated and propagated throughout the world.

This command is primarily a diagnostic and administrative tool. It allows operators to dynamically switch the active lighting algorithm at runtime, typically between a standard, computationally intensive algorithm (Flood) and a simplified, performance-oriented one (FullBright). This capability is critical for debugging lighting artifacts, performance profiling, or temporarily disabling complex lighting for specific server events.

By extending AbstractWorldCommand, it inherits the necessary boilerplate to ensure it operates within the context of a specific game world and has access to its state, enforcing a clean separation of concerns between command parsing and world state mutation.

### Lifecycle & Ownership
- **Creation:** A single instance of LightingCalculationCommand is instantiated by the server's command registration system during the initial server bootstrap sequence. It is registered under the name *calculation*.
- **Scope:** The object is a singleton that persists for the entire server session. Its state, which consists of its argument definitions, is configured once at creation and never changes.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** The internal state of a LightingCalculationCommand instance is **immutable** after construction. It holds final references to its argument parsers (calculationTypeArg, invalidateFlag) and pre-compiled message templates. It does not cache any world data or maintain any state between executions. Each call to the execute method is independent and idempotent given the same world state and arguments.

- **Thread Safety:** The object itself is inherently thread-safe due to its immutable state. However, its primary method, execute, performs direct mutations on the World and ChunkLightingManager objects. These world-level systems are **not thread-safe**.

    **Warning:** The execute method must *only* be invoked from the main server thread (the world tick thread). The command system guarantees this contract. Any attempt to invoke this method from an asynchronous task or external thread will lead to severe concurrency issues, including data corruption and server instability.

## API Surface
The public contract is defined by its role as a command. Direct invocation outside the command system is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) / O(N) | Executes the command logic. Complexity is O(1) for switching the calculation strategy but becomes O(N) if the *invalidate* flag is used, where N is the number of currently loaded chunks. This triggers a potentially expensive relighting operation across all chunks. |

## Integration Patterns

### Standard Usage
Developers do not instantiate or call this class directly. Instead, an instance is registered with the command system. The system then invokes it based on user input.

```java
// Example of how the system registers this command during startup
// (Conceptual - actual implementation may vary)

CommandRegistry registry = server.getCommandRegistry();
CommandNode lightingNode = registry.getNode("lighting");

// The command is instantiated once and registered
lightingNode.registerSubCommand(new LightingCalculationCommand());
```

A user with appropriate permissions then triggers its execution via the server console or in-game chat.

> `/lighting calculation FULLBRIGHT --invalidate`

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually to call its methods. The command system is responsible for its lifecycle. Bypassing the system breaks context-aware features and safety guarantees.
- **Asynchronous Execution:** Do not acquire a reference to this command and call its execute method from a separate thread. All world modifications must be synchronized with the main server tick.

## Data Pipeline
The flow of data and control for this command is linear and initiated by an external user action. The command translates a high-level user request into a low-level, direct modification of a core engine system.

> Flow:
> User Command Input -> Command System Parser -> **LightingCalculationCommand.execute** -> World.getChunkLighting() -> ChunkLightingManager.setLightCalculation() -> World State Mutation -> (Optional) ChunkLightingManager.invalidateLoadedChunks() -> Asynchronous Chunk Relighting Task


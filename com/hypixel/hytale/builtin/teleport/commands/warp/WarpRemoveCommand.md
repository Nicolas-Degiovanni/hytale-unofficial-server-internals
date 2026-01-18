---
description: Architectural reference for WarpRemoveCommand
---

# WarpRemoveCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.warp
**Type:** Transient

## Definition
```java
// Signature
public class WarpRemoveCommand extends CommandBase {
```

## Architecture & Concepts
The WarpRemoveCommand class is a command handler within the server's Command System. It serves as the specific implementation for the user-facing action of deleting a named warp location. This class is a terminal component in the command processing chain; it does not delegate to other commands but instead orchestrates operations across multiple core engine systems.

Architecturally, this command acts as a transactional controller that bridges three distinct domains:
1.  **The Command System:** It registers itself with a specific name ("remove") and argument structure, allowing the system to route player or console input to its execution logic.
2.  **The Plugin Data Layer:** It interacts directly with the TeleportPlugin singleton to modify the persistent list of defined warps. This represents the logical data model for warps.
3.  **The Entity Component System (ECS):** It schedules a task on the target World to find and remove the in-game entity associated with the warp. This represents the physical, in-world manifestation of the warp data.

This design enforces a strong separation of concerns. The command itself is stateless, the plugin manages the canonical warp data, and the World manages the corresponding game entities.

### Lifecycle & Ownership
-   **Creation:** A single instance of WarpRemoveCommand is instantiated by the server's command registration system during the bootstrap phase of the TeleportPlugin. It is discovered via reflection or explicit registration within the plugin's entry point.
-   **Scope:** The object instance is a long-lived singleton managed by the command registry, persisting for the entire server session. However, its *execution context* is ephemeral, created and destroyed for each individual invocation of the command.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the TeleportPlugin is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is effectively stateless and immutable after its initial construction. Its internal fields consist of final Message constants and a final RequiredArg definition. All stateful operations are performed on external, long-lived objects, primarily the TeleportPlugin and the target World's EntityStore.

-   **Thread Safety:** **This class is not thread-safe.** The primary execution method, executeSync, is explicitly designed to be invoked by the Command System on the main server thread. It performs non-atomic operations on the shared state of TeleportPlugin and schedules subsequent work on a specific World's thread. Any attempt to invoke executeSync from a concurrent thread will result in severe data corruption, race conditions, and server instability. The framework's design guarantees safe, synchronous execution.

## API Surface
The public contract of this class is fulfilled by its interaction with the command framework, not through direct developer invocation. The primary method is the protected override of executeSync.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Executes the warp removal logic. This is the sole entry point for the command's functionality. Complexity is dominated by the parallel entity scan, making it O(N) where N is the number of entities with a WarpComponent in the target world. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or invocation by developers. It is automatically registered and executed by the server's command handling system. A user or administrator triggers its execution by typing the command in-game or in the console.

**Example User Interaction:**
```
/warp remove MyCoolWarp
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new WarpRemoveCommand()`. The command system manages the lifecycle of command objects. Direct instantiation results in an object that is not registered to handle any commands.
-   **Manual Invocation:** Do not acquire an instance of this command and call `executeSync` manually. This bypasses the entire command framework, including critical permission checks, argument parsing and validation, and thread safety guarantees.

## Data Pipeline
The execution of this command initiates a one-way data and control flow that permanently alters server state. The flow guarantees that the logical warp record is removed before its corresponding in-world entity is deleted.

> Flow:
> Player Command Input -> Server Command Parser -> **WarpRemoveCommand.executeSync** -> TeleportPlugin.getWarps().remove() -> TeleportPlugin.saveWarps() -> Universe.getWorld() -> World.execute(task) -> EntityStore.forEachEntityParallel() -> CommandBuffer.removeEntity()


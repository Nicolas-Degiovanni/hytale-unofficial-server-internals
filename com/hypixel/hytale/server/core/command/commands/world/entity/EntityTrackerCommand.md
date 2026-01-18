---
description: Architectural reference for EntityTrackerCommand
---

# EntityTrackerCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityTrackerCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The EntityTrackerCommand is a server-side diagnostic command designed to provide administrators with a real-time snapshot of the Entity Tracking System's state for a specific player. It serves as a read-only interface into the complex visibility and culling calculations performed by the server for each client.

Architecturally, this class is a terminal node in the server's Command System. It is invoked by the command dispatcher and given direct, albeit temporary, access to a specific World's core Entity-Component-System (ECS) data store. Its primary function is to query ECS components related to entity visibility—specifically the EntityTrackerSystems.EntityViewer and Player components—and format this low-level data into a human-readable summary.

This command is critical for debugging issues related to entity spawning, despawning, and Level of Detail (LOD) transitions from a player's perspective. It does not perform any calculations itself; it merely reports the results of the underlying EntityTrackerSystems.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityTrackerCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is discovered and added to a central command registry.
- **Scope:** The command object persists for the entire server session, held as a reference within the global command registry. It is stateless between executions.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively immutable after its initial construction. It contains a final field, playerArg, which defines the command's argument structure. No state is stored or modified between calls to the execute method. All necessary state is provided via method parameters (CommandContext, World, Store).
- **Thread Safety:** The execute method is not thread-safe and must be invoked from the main world thread. The command system guarantees this execution context. Direct, multi-threaded invocation of the execute method will lead to race conditions and concurrent modification exceptions when accessing the ECS Store, which is not designed for concurrent access from arbitrary threads.

## API Surface
The public contract is fulfilled by its constructor for registration and the overridden execute method for invocation by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the command logic. Queries the ECS Store for entity tracking data for a target player and sends a summary message to the command source. Throws exceptions if arguments are null. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically. It is designed for invocation by a server administrator or a player with sufficient permissions via the in-game chat or server console.

```text
// Example of in-game invocation
/tracker Steve
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create instances of this class using `new EntityTrackerCommand()`. The server's command system manages the lifecycle of all registered commands. Manual instantiation bypasses registration and will result in a non-functional object.
- **Manual Execution:** Avoid calling the `execute` method directly. Always dispatch commands through the central `CommandManager` or equivalent service. This ensures the `CommandContext` is correctly constructed and that permission checks are enforced.
- **State Assumption:** Do not assume the data retrieved by this command is static. It represents a single moment in time. Running the command again, even a single tick later, may produce different results due to the dynamic nature of the entity tracking system.

## Data Pipeline
The command acts as a query endpoint in a larger data flow, translating a user's request into a view of internal server state.

> Flow:
> User Input (`/tracker <player>`) -> Network Layer -> Command Parser -> **EntityTrackerCommand.execute()** -> ECS Store Query -> Message Builder -> Formatted Message -> CommandContext -> Network Layer -> Client Chat UI

---


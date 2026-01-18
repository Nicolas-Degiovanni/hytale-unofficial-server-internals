---
description: Architectural reference for EntityResendCommand
---

# EntityResendCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Singleton (within Command Registry)

## Definition
```java
// Signature
public class EntityResendCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The EntityResendCommand is a concrete implementation within the server's Command System framework. It serves as a powerful administrative and debugging tool for resolving client-side entity desynchronization issues.

Architecturally, this command does not directly manipulate network packets or entity data. Instead, it acts as a control signal to a more complex underlying system: the EntityTrackerSystems. Its primary function is to invalidate a specific player's view of the world's entities.

The core operational concept is **resynchronization through invalidation**. By invoking EntityTrackerSystems.despawnAll, the command effectively tells the server to "forget" every entity it has sent to the target player. On subsequent server ticks, the entity tracking system will re-evaluate the player's position and perception sphere, naturally re-sending all necessary entity creation packets. This ensures the player's client receives a completely fresh and accurate state of all nearby entities.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityResendCommand is created by the server's command registration service during the initial server bootstrap phase. It is discovered and registered alongside all other server commands.
- **Scope:** The object instance persists for the entire server session, held in memory by the central command registry. It is not created per-execution.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The EntityResendCommand object is effectively stateless concerning individual executions. It holds a final, immutable field, playerArg, which defines the command's argument structure. All state required for execution (the target player, the world, the entity store) is passed into the execute method via the CommandContext.
- **Thread Safety:** This class is **not thread-safe** and is not designed to be. Command execution is expected to be managed by the server's main game loop, which processes commands sequentially on the primary world thread. This guarantees that its operations on the World and EntityStore are atomic relative to other game logic.

**WARNING:** Invoking the execute method from an asynchronous thread will lead to severe concurrency violations, data corruption, and server instability.

## API Surface
The public contract is defined by its role as a command and is limited to its constructor and the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N) | Executes the command's logic. N is the number of entities currently tracked for the target player. Triggers a full despawn of all entities for the player specified in the context. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly via Java code. It is designed to be invoked by a server administrator or through an automated system via the server console or in-game chat.

```text
# Example of a server administrator invoking the command
/resend Notch
```
The command system parses this input, identifies the EntityResendCommand, resolves the PlayerRef argument "Notch", and invokes the execute method with the appropriate context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityResendCommand()` in game logic. The command system relies on a single, registered instance.
- **Manual Invocation:** Avoid calling the execute method directly. All commands should be dispatched through the server's central command service to ensure proper permission checks, argument parsing, and execution context.
- **Stateful Logic:** Do not modify this class to hold state related to its execution. This would break its singleton nature and introduce race conditions if the command system were ever to become multi-threaded.

## Data Pipeline
EntityResendCommand initiates a control flow rather than a data processing flow. It triggers a state reset in a downstream system, which in turn generates a new data flow to the client.

> Flow:
> Administrator Input (`/resend <player>`) -> Server Command Parser -> **EntityResendCommand.execute()** -> EntityTrackerSystems.despawnAll() -> (Next Tick) Entity Tracker Re-sends Entities -> Network Subsystem -> Client Render State Update


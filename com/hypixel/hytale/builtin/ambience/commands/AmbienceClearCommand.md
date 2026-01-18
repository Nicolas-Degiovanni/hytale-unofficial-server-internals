---
description: Architectural reference for AmbienceClearCommand
---

# AmbienceClearCommand

**Package:** com.hypixel.hytale.builtin.ambience.commands
**Type:** Transient

## Definition
```java
// Signature
public class AmbienceClearCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The AmbienceClearCommand class is a server-side implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. It provides a direct, user-invokable interface for administrators to manipulate the ambient music state of a specific game world.

Its primary architectural role is to act as a transactional bridge between a user command (e.g., a chat message) and the underlying world state. By extending AbstractWorldCommand, it inherits the necessary boilerplate for world and entity store context injection, ensuring that its logic operates on the correct world instance from which the command was issued. This class is a leaf node in the command hierarchy, responsible for a single, atomic operation: clearing the forced ambient music track from a world's AmbienceResource.

## Lifecycle & Ownership
- **Creation:** A single instance of AmbienceClearCommand is instantiated by the server's CommandSystem during the server bootstrap and command registration phase. It is not created on-demand per execution.
- **Scope:** The object instance persists for the entire server session, held within the command registry. The context, world, and store objects passed to its execute method are transient and scoped only to that specific invocation.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the CommandSystem is dismantled.

## Internal State & Concurrency
- **State:** AmbienceClearCommand is fundamentally stateless. It contains no mutable fields and all necessary state (the target world, the command sender) is provided via method arguments during the execution call. This design ensures that a single instance can be safely reused for all executions of the command across the server.
- **Thread Safety:** The class itself is immutable and therefore thread-safe. However, the operations it performs within the execute method are subject to the concurrency guarantees of the underlying world and resource systems. The command system typically serializes command execution per-world, but developers should not assume that the AmbienceResource itself is safe for concurrent modification from other systems.

## API Surface
The public contract is defined by its parent, AbstractWorldCommand. The core logic resides in the protected execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | protected void | O(log N) | Retrieves the AmbienceResource from the world's store and sets the forced music to null. Complexity is dominated by the resource lookup in the store. Sends a success message to the command originator. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked directly from code. It is designed to be executed by the server's command handler in response to user input.

A server administrator or a player with appropriate permissions would execute the command via the game's chat console.

```
/ambience clear
```
This input is parsed by the server, which identifies the AmbienceClearCommand, validates permissions, and invokes its execute method with the appropriate world context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AmbienceClearCommand()`. The command system handles instantiation and registration. A manually created instance will not be executable within the server environment.
- **Manual Execution:** Calling the `execute` method directly from other game logic is a severe anti-pattern. This bypasses critical infrastructure, including permission checks, context validation, and command dispatching queues. To modify the ambience state programmatically, interact with the AmbienceResource API directly.

## Data Pipeline
The command facilitates a simple, one-way data flow to modify world state, with a feedback message sent to the user.

> Flow:
> User Input (`/ambience clear`) -> Server Command Parser -> **AmbienceClearCommand.execute()** -> World.EntityStore.getResource() -> AmbienceResource.setForcedMusicAmbience(null) -> CommandContext.sendMessage() -> Client UI


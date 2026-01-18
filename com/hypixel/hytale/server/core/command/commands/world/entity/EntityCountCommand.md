---
description: Architectural reference for EntityCountCommand
---

# EntityCountCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityCountCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The EntityCountCommand is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. It encapsulates a single, specific server action: retrieving and reporting the total number of entities within a given world.

By extending AbstractWorldCommand, this class delegates the responsibility of context validation to its parent. The framework ensures that a valid World and its associated EntityStore are resolved and provided before the execute method is ever called. This design decouples the command's core logic from the complexities of world management and player session handling, resulting in a highly focused and testable component.

This command is purely a data-retrieval operation. It does not modify world state and serves as a simple diagnostic tool for server administrators.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the command registration service during server bootstrap. The system likely scans the classpath for command implementations and instantiates them to build a registry.
- **Scope:** The object instance persists for the entire server session, held within the central command registry. It is effectively a singleton from the perspective of the command system.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or if the command is dynamically unregistered.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no mutable fields and its behavior is determined entirely by the arguments passed to its execute method. All state it interacts with, such as the entity count, is owned by the EntityStore.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the command system guarantees that the execute method is invoked on the main server thread for the target world. This is a critical design constraint to prevent race conditions when accessing the underlying EntityStore, which is not guaranteed to be safe for concurrent access from other threads.

## API Surface
The public contract is defined by its parent, AbstractWorldCommand. The primary entry point is the execute method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Fetches the entity count from the provided store and sends it as a formatted message to the command's originator. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. It is invoked by the command system in response to user input. An administrator or a player with sufficient permissions would execute the command from the game client or server console.

**User Interaction Example:**
```
/entity count
```
This input is parsed by the server, which identifies the "entity count" command, resolves the EntityCountCommand handler, and invokes its execute method with the appropriate context for the user's current world.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new EntityCountCommand()` in application logic. The command system is responsible for the object's lifecycle. Direct invocation bypasses permission checks, context resolution, and argument parsing.
- **Stateful Implementation:** Do not add mutable fields to this class. A single instance is shared across all invocations, and adding state would introduce severe concurrency bugs and unpredictable behavior.

## Data Pipeline
The flow of data for a typical command execution is linear, passing from the user, through the server's command processing stack, to this class, and back to the user.

> Flow:
> Player Input -> Network Packet -> Server Command Parser -> **EntityCountCommand.execute()** -> EntityStore.getEntityCount() -> Message Translation System -> CommandContext -> Network Packet -> Player Chat UI


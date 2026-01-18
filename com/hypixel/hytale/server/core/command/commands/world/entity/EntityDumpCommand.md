---
description: Architectural reference for EntityDumpCommand
---

# EntityDumpCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityDumpCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The EntityDumpCommand is a server-side diagnostic tool integrated into the command system. Its primary purpose is to provide developers and server administrators with a low-level snapshot of an in-game entity's state. It achieves this by serializing the entity's complete component data into a BSON document and logging it as a human-readable JSON string.

This class extends AbstractWorldCommand, signifying that its operations are scoped to a specific World instance and require access to its underlying data stores. It acts as a direct interface to the EntityStore, the core storage layer for all entity data within the game universe. The command leverages the argument parsing framework, specifically ArgTypes.ENTITY_ID, to resolve a user-provided identifier or the entity in the user's line of sight into a valid entity reference.

This command is critical for debugging complex entity behaviors, state corruption, or unexpected component values without attaching a debugger.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityDumpCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is not created on a per-execution basis.
- **Scope:** The object's lifecycle is tied to the server's lifecycle. It persists from the moment it is registered until the server shuts down.
- **Destruction:** The instance is eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The instance field entityArg is configured once in the constructor and is treated as immutable thereafter. All state required for execution, such as the target world and the command issuer, is passed into the execute method via the CommandContext.
- **Thread Safety:** This class is not thread-safe. The execute method is designed to be called exclusively from the main server thread, which has safe access to the World and EntityStore. Invoking this method from a worker thread would lead to severe concurrency violations, data corruption, and server instability. The underlying game state that it accesses is not thread-safe.

## API Surface
The public contract is defined by its parent, AbstractWorldCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(S) | Executes the dump operation. S represents the size of the target entity's component data. This method reads from the EntityStore, serializes the data, and writes to the log. It sends a feedback message to the user via the CommandContext. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a privileged user through the server console or in-game chat. The command system handles parsing, permission checks, and invocation.

```java
// This code is conceptual and represents the server's internal dispatching.
// A developer would not write this.
CommandDispatcher dispatcher = server.getCommandDispatcher();
dispatcher.dispatch("dump", commandSource); // Dumps entity in view

dispatcher.dispatch("dump 12345", commandSource); // Dumps entity with ID 12345
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityDumpCommand()`. The command will not be registered with the server's command system and will be non-functional. Command objects must be managed by the server's registration lifecycle.
- **Manual Invocation:** Do not call the `execute` method directly. This bypasses critical infrastructure for argument parsing, context creation, and thread safety guarantees. Manually constructing the required World and Store parameters is complex and highly likely to cause state-related errors.

## Data Pipeline
The flow of data for this command is a one-way diagnostic pipeline from the game world to the server logs.

> Flow:
> User Input (`/dump`) -> Command System Parser -> **EntityDumpCommand.execute** -> EntityStore Query -> Entity Data Copy -> BSON Serialization -> JSON Conversion -> HytaleLogger -> Server Log File & Console Output


---
description: Architectural reference for EntityCommand
---

# EntityCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Singleton

## Definition
```java
// Signature
public class EntityCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The EntityCommand class is a **Command Aggregator**. It does not implement any executable logic itself. Instead, its sole responsibility is to serve as the root namespace for a collection of related sub-commands that operate on game world entities. This class acts as a structural component within the server's command processing system.

By extending AbstractCommandCollection, it inherits the necessary machinery to register and dispatch commands. When a command like `/entity count` is executed, the server's central CommandManager first identifies the root token *entity* and routes the request to the singleton instance of this EntityCommand. The parent AbstractCommandCollection logic then parses the subsequent token, *count*, and dispatches the execution to the registered EntityCountCommand instance.

This design follows the **Composite Pattern**, allowing the server to treat a complex tree of commands as a single, unified entry point, which simplifies registration and discovery.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's core CommandManager during the server bootstrap sequence. The system scans for and registers all top-level command collections like this one.
- **Scope:** Application-scoped. The single instance of EntityCommand persists for the entire lifetime of the server process.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server shuts down and the CommandManager is cleared. There is no explicit destruction or cleanup logic.

## Internal State & Concurrency
- **State:** **Effectively Immutable**. The internal list of sub-commands is populated exclusively within the constructor. Once the object is constructed, its state does not change for the remainder of its lifecycle. It holds no runtime data or caches.
- **Thread Safety:** **Thread-Safe**. As the internal state is immutable post-construction, this class can be safely accessed from multiple threads without synchronization. Any concurrency concerns related to command execution are managed by the CommandManager and the individual sub-command implementations, not by this container class.

## API Surface
The public contract of this class is minimal and is primarily concerned with its construction. All other relevant behaviors are inherited from AbstractCommandCollection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntityCommand() | constructor | O(N) | Constructs the command group. Instantiates and registers N sub-commands related to entity management. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic interaction. Its usage is implicit through the server's command system. A server administrator or a player with sufficient permissions would use it via the server console or in-game chat.

```
# Example command entered by a user
/entity count

# Another example with arguments
/entity clone <source_entity_selector> <target_position>
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityCommand()` in application code. The command system is responsible for the lifecycle of all command objects. Manually creating an instance will result in an orphaned object that is not registered to handle any input.
- **Post-Construction Modification:** Do not attempt to retrieve this object from the command registry to add or remove sub-commands at runtime. The command hierarchy is designed to be static and defined at server startup.

## Data Pipeline
The EntityCommand class acts as a router in the data flow of command processing. It receives a command request and dispatches it to the appropriate handler.

> Flow:
> Player/Console Input -> Server Network Listener -> CommandManager -> **EntityCommand** (Routing Logic) -> Specific Sub-Command (e.g., EntityCountCommand) -> World State Modification


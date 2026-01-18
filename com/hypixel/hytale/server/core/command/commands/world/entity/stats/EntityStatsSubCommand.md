---
description: Architectural reference for EntityStatsSubCommand
---

# EntityStatsSubCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.stats
**Type:** Transient

## Definition
```java
// Signature
public class EntityStatsSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The EntityStatsSubCommand class is a structural component within the server's command processing system. It embodies the *Composite* design pattern, acting as a non-terminal node in the command tree. Its sole architectural purpose is to group and route related sub-commands under a unified namespace, in this case, `stats`.

This class contains no direct execution logic. When a user issues a command such as `/entity <target> stats get ...`, the primary command parser first identifies and delegates control to the parent `entity` command. The `entity` command's parser then consumes the `stats` token and passes the remainder of the command string to this EntityStatsSubCommand instance for further routing. This instance, in turn, matches the next token (e.g., `get`) to one of its registered child commands, such as EntityStatsGetCommand, which finally executes the requested action.

This hierarchical design is critical for maintaining a clean, scalable, and discoverable command structure. It prevents pollution of the global command namespace and provides a logical grouping for related server administration functions.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by its parent command collection (e.g., an `EntityCommand`) during the server's bootstrap sequence when the central CommandManager registers all core commands.
-   **Scope:** The object's lifecycle is bound to the server's CommandManager. It persists for the entire server session and is part of the static command tree.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only upon server shutdown when the CommandManager and its entire command tree are dismantled.

## Internal State & Concurrency
-   **State:** This object is effectively **immutable** after its constructor completes. Its internal state is a list of child command objects, which is populated once and never modified during runtime. It holds no session-specific data, caches, or mutable fields.
-   **Thread Safety:** The class is inherently **thread-safe**. Due to its immutable nature, multiple threads can safely traverse this node in the command tree simultaneously without locks or synchronization primitives. This is essential in a multi-user server environment where many commands may be processed concurrently.

## API Surface
This class has no public API intended for general consumption. Its interaction with the system is declarative, defined by its construction and registration within a parent command collection. The methods inherited from AbstractCommandCollection, such as `addSubCommand`, are exclusively used internally during the initial setup of the command hierarchy.

## Integration Patterns

### Standard Usage
This class is not meant to be invoked directly. It is a structural element used to build the command tree. A parent command would register it as a child to handle a specific sub-domain of commands.

```java
// Example from a hypothetical parent command's constructor
// This is how EntityStatsSubCommand is integrated into the system.
public class EntityCommand extends AbstractCommandCollection {
    public EntityCommand() {
        super("entity", "...");
        
        // Registering the collection to handle the "stats" subcommand
        this.addSubCommand(new EntityStatsSubCommand());
        
        // ... other subcommands like "teleport", "inventory", etc.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never instantiate this class for any purpose other than registering it with a parent command collection. It provides no standalone functionality.
-   **Runtime Modification:** Do not attempt to dynamically add or remove sub-commands from this collection after server initialization. The command tree is designed to be static for performance and predictability. Modifying it at runtime can lead to unpredictable behavior and concurrency issues.

## Data Pipeline
EntityStatsSubCommand acts as a routing node in the server's command control flow. It does not process game data directly but rather directs the flow of command execution based on user input.

> Flow:
> Raw Command String -> CommandManager Parser -> Parent Command (e.g., EntityCommand) -> **EntityStatsSubCommand** -> Child Command (e.g., EntityStatsGetCommand) -> Game State Read/Write


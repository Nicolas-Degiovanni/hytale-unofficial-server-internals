---
description: Architectural reference for EntitySnapshotSubCommand
---

# EntitySnapshotSubCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.snapshot
**Type:** Transient

## Definition
```java
// Signature
public class EntitySnapshotSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The EntitySnapshotSubCommand class is not a command that executes an action itself. Instead, it functions as a structural component within the server's command processing system, specifically as a **sub-command group**. It implements the **Composite Design Pattern** to create a hierarchical command tree.

Its primary role is to act as a routing node, grouping related entity snapshot commands (like *length* and *history*) under a single, logical namespace: `snapshot`. When the command dispatcher processes a command like `/entity snapshot history`, it first routes to the parent `entity` command, which then delegates control to this `EntitySnapshotSubCommand` instance. This class then inspects the subsequent arguments to dispatch control to the appropriate child command, such as EntitySnapshotHistoryCommand.

This architectural pattern avoids monolithic command classes and promotes modular, organized, and extensible command structures.

### Lifecycle & Ownership
- **Creation:** Instantiated once during the server bootstrap sequence. It is created and registered by its parent command collection, likely a top-level `EntityCommand`. It is not created on-demand per command execution.
- **Scope:** The instance persists for the entire server session, held as a child within the server's central command registry graph.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server shuts down and the command registry is dismantled.

## Internal State & Concurrency
- **State:** This class is effectively **immutable** post-construction. Its internal state consists of a list of sub-commands which is populated exclusively within the constructor and is not modified during the server's operational lifetime. It holds no per-execution state.
- **Thread Safety:** The class is inherently **thread-safe**. As its internal structure is immutable after initialization, it can be safely traversed by multiple threads processing different commands concurrently without locks or synchronization. Responsibility for thread safety during command execution is delegated to the final leaf-node command that is executed.

## API Surface
The public contract of this class is minimal and primarily declarative. Interaction is not meant to occur through direct method calls but through its registration with the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntitySnapshotSubCommand() | Constructor | O(1) | Constructs the command group. Registers the primary name "snapshot", the alias "snap", and populates its internal collection with child commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be registered as a sub-command within a parent AbstractCommandCollection, typically during server initialization.

```java
// Example from a hypothetical parent command (e.g., EntityCommand)
public class EntityCommand extends AbstractCommandCollection {
    public EntityCommand() {
        super("entity", "...");
        
        // Correct Integration: Add an instance as a sub-command
        this.addSubCommand(new EntitySnapshotSubCommand());
        
        // ... add other entity-related sub-commands
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Creating an instance via `new EntitySnapshotSubCommand()` without adding it to a parent command collection is pointless. The instance will be immediately garbage collected and will not be part of the executable command tree.
- **Manual Execution:** Do not attempt to call execution methods inherited from parent classes. This class is a router; the server's command dispatcher is solely responsible for invoking the correct execution chain.

## Data Pipeline
This component operates on the control flow of command parsing, not a data processing pipeline. It directs execution based on user input.

> Flow:
> Raw Command String -> Command Parser -> Parent Command (`EntityCommand`) -> **`EntitySnapshotSubCommand`** -> Child Command (`EntitySnapshotHistoryCommand`) -> Command Executor


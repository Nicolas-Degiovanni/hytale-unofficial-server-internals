---
description: Architectural reference for PrefabPathUpdateCommand
---

# PrefabPathUpdateCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPathUpdateCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PrefabPathUpdateCommand class is a structural component within the server's command processing system. It does not implement any direct game logic. Instead, it functions as a **command collection**, a container that groups related subcommands under a single, shared namespace.

In this case, it establishes the `update` branch within a larger command tree, likely for managing NPC paths. It acts as an intermediary node that delegates execution to one of its registered children, such as PrefabPathUpdatePauseCommand or PrefabPathUpdateObservationAngleCommand, based on the subsequent arguments provided by the user.

This design follows the **Composite Pattern**, allowing the command system to treat both individual command objects and collections of commands uniformly. This simplifies the parsing and dispatching logic by creating a hierarchical command tree.

### Lifecycle & Ownership
- **Creation:** An instance is created and registered during the server's command system bootstrap phase. It is instantiated by its parent command collection, which then adds it to its own list of subcommands.
- **Scope:** The object's lifecycle is tied to the server's command registry. It persists for the entire duration of the server session.
- **Destruction:** The instance is eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively **immutable** after its constructor completes. It holds a list of its subcommands, but this list is populated once during instantiation and is not modified thereafter. It maintains no other runtime state.
- **Thread Safety:** The class is inherently **thread-safe**. Its immutable nature ensures that it can be safely accessed by the command dispatcher from any thread without requiring synchronization or locks.

## API Surface
The public contract is limited to its constructor, as the class is designed for declarative registration, not programmatic invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PrefabPathUpdateCommand() | Constructor | O(1) | Instantiates the command group, registering the name "update" and adding its child subcommands to the internal collection. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in game logic. Its integration is declarative, as part of the server's command definitions. A parent command would instantiate and register it.

```java
// Hypothetical parent command registering this collection
public class NpcPathCommand extends AbstractCommandCollection {
    public NpcPathCommand() {
        super("npcpath", "...");
        
        // Standard integration: add as a subcommand
        this.addSubCommand(new PrefabPathUpdateCommand());
        this.addSubCommand(new NpcPathCreateCommand());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Execution:** Never create an instance of this class to try and "run" a command. The server's command dispatcher is solely responsible for parsing input and routing it through the command tree.
- **Post-Construction Modification:** Do not attempt to retrieve an instance of this class from the command registry to add or remove subcommands at runtime. The command hierarchy is designed to be static after server initialization.

## Data Pipeline
This class acts as a routing node in the command processing pipeline. It does not transform data but directs the flow based on command arguments.

> Flow:
> Raw Command String -> Command Dispatcher -> Parent Command -> **PrefabPathUpdateCommand** -> (Dispatch to Subcommand) -> PrefabPathUpdatePauseCommand


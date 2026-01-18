---
description: Architectural reference for WorldMapViewRadiusSubCommand
---

# WorldMapViewRadiusSubCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapViewRadiusSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The WorldMapViewRadiusSubCommand class is a structural component within the server's command handling framework. It embodies the **Composite Pattern**, acting as a non-terminal node in the command tree. Its sole responsibility is to group related sub-commands—specifically Get, Set, and Remove—under a single, logical namespace: *viewradius*.

This class contains no business logic. When the command dispatcher processes a command like `/worldmap viewradius set ...`, it first routes the request to the parent `WorldMapCommand`, which then delegates to this `WorldMapViewRadiusSubCommand` instance. This object, in turn, parses the next argument ("set") and forwards the execution context to the corresponding child command, `WorldMapViewRadiusSetCommand`.

This architectural approach is critical for creating a clean, hierarchical, and extensible command-line interface. It prevents pollution of the global command namespace and improves discoverability for server administrators.

### Lifecycle & Ownership
- **Creation:** Instantiated once during server initialization. It is constructed and immediately registered as a child of a parent command, likely a `WorldMapCommand`, as part of the command registry bootstrap process.
- **Scope:** Application-scoped. It persists for the entire lifetime of the server's command registry.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection only when the command registry is cleared, typically during a server shutdown or a full command system reload.

## Internal State & Concurrency
- **State:** Effectively **Immutable** post-construction. The collection of sub-commands is populated within the constructor and is not designed to be modified at runtime. This class holds no dynamic or session-specific state.
- **Thread Safety:** This class is inherently **Thread-Safe**. Its immutable nature allows multiple threads to traverse it without locks or synchronization. Responsibility for thread safety during command execution is delegated to the terminal leaf commands (e.g., `WorldMapViewRadiusSetCommand`), which perform the actual state-mutating work.

## API Surface
This class has no meaningful public API intended for direct invocation by developers. Its interaction with the system is declarative, defined by its registration into a parent command collection. The primary public contract is the command name "viewradius" that it registers.

## Integration Patterns

### Standard Usage
A developer does not interact with an instance of this class directly. Instead, it is registered declaratively within a parent command's constructor.

```java
// Example from a hypothetical parent command
public class WorldMapCommand extends AbstractCommandCollection {
    public WorldMapCommand() {
        super("worldmap", "...");
        
        // Correctly register this collection as a sub-command
        this.addSubCommand(new WorldMapViewRadiusSubCommand());
        // ... other sub-commands
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation without Registration:** Creating `new WorldMapViewRadiusSubCommand()` without adding it to a parent command collection renders the object useless, as it will never be reachable by the command dispatcher.
- **Adding Business Logic:** Do not add execution logic to this class. It is designed strictly as a container. All logic must be encapsulated in the final leaf commands (`...GetCommand`, `...SetCommand`). Modifying this class to perform actions would violate the Composite Pattern and the Single Responsibility Principle.

## Data Pipeline
This component acts as a router in a control-flow pipeline, not a data-processing pipeline. It directs the flow of command execution based on user input.

> Flow:
> User Input (`/worldmap viewradius set 10`) -> Command Dispatcher -> `WorldMapCommand` -> **`WorldMapViewRadiusSubCommand`** -> `WorldMapViewRadiusSetCommand` -> Execution & State Change


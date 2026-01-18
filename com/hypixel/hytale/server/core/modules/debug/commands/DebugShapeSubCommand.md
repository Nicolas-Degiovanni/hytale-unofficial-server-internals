---
description: Architectural reference for DebugShapeSubCommand
---

# DebugShapeSubCommand

**Package:** com.hypixel.hytale.server.core.modules.debug.commands
**Type:** Transient

## Definition
```java
// Signature
public class DebugShapeSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The DebugShapeSubCommand class is an implementation of the **Composite Pattern** within the server's command processing framework. It functions as a structural grouping mechanism, not an executable command in itself. Its primary architectural role is to organize and namespace a suite of related debugging commands under a single, coherent entry point: *shape*.

This class acts as an intermediary routing node in the command tree. When a command like `/debug shape sphere` is issued, the core command system first routes the request to the parent `debug` command, which then delegates to this DebugShapeSubCommand instance. This class, in turn, parses the next argument (*sphere*) and dispatches the request to the corresponding registered sub-command, such as DebugShapeSphereCommand.

This design decouples the parent command from the specific implementations of its children, allowing new debug shapes to be added without modifying higher-level command collections.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor within a parent AbstractCommandCollection, typically the primary `DebugCommand`. This occurs once during the server's command registration phase at startup. It is not managed by a dependency injection container.
- **Scope:** The object's lifetime is bound to the server's command registry. It persists for the entire server session, from the moment it is registered until the server shuts down or the command module is unloaded.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the parent command collection is unregistered or the entire command registry is cleared.

## Internal State & Concurrency
- **State:** The state of this object is effectively **immutable** after construction. The internal list of sub-commands is populated within the constructor and is not designed to be modified at runtime. All state related to command parameters and execution context is passed in during method invocation and is not stored in instance fields.
- **Thread Safety:** This class is inherently thread-safe. Command execution is managed by a dedicated, single-threaded command processor within the server. As this class holds no mutable state, concurrent access poses no risk.

**WARNING:** While this collection class is thread-safe, the sub-commands it invokes (e.g., DebugShapeSphereCommand) may interact with non-thread-safe game systems. The overall safety of a command execution is determined by the final concrete command implementation, not this routing class.

## API Surface
The public API of this class is minimal, as its primary interaction is through composition and inheritance. The constructor is the main entry point for its configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DebugShapeSubCommand() | Constructor | O(k) | Instantiates the command collection and registers a fixed set of k sub-commands for various shapes. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be added as a sub-command to a parent collection, establishing a command hierarchy.

```java
// Example from a hypothetical parent command collection
// This is the ONLY correct way to integrate this class.

public class DebugCommand extends AbstractCommandCollection {
    public DebugCommand() {
        super("debug", "Root command for all debug operations.");

        // Register the shape command group
        this.addSubCommand(new DebugShapeSubCommand());

        // Register other debug command groups
        this.addSubCommand(new DebugEntitySubCommand());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Execution:** Do not create an instance of this class to try and execute a command programmatically. The command system handles the entire lifecycle.
- **Runtime Modification:** Do not attempt to retrieve an instance of this class from the command registry to add or remove sub-commands after initialization. The command hierarchy is intended to be static.

## Data Pipeline
DebugShapeSubCommand serves as a routing step in the server's command processing pipeline. It receives a partially parsed command context and dispatches it to the next appropriate handler based on the command arguments.

> Flow:
> Player Chat Input (`/debug shape sphere ...`) -> Network Layer -> Command Parser -> `DebugCommand` -> **`DebugShapeSubCommand`** -> `DebugShapeSphereCommand` -> World State / Rendering System


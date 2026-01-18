---
description: Architectural reference for HitboxCollisionCommand
---

# HitboxCollisionCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.component.hitboxcollision
**Type:** Transient

## Definition
```java
// Signature
public class HitboxCollisionCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The HitboxCollisionCommand class is a *Command Group* within the server's command processing system. It functions as a structural component, designed to aggregate and route related debug functionalities under a single, unified command namespace.

This class does not implement any direct game logic. Its sole responsibility is to serve as a parent for its registered sub-commands: HitboxCollisionAddCommand and HitboxCollisionRemoveCommand. By inheriting from AbstractCommandCollection, it leverages the framework's machinery for parsing command strings and dispatching execution to the appropriate sub-command handler.

This architectural pattern, known as the Composite pattern, simplifies the command hierarchy and improves usability for server administrators by grouping related actions (e.g., `/hitboxcollision add`, `/hitboxcollision remove`) under a common entry point.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central CommandManager during the initial bootstrap or plugin loading phase. This is typically part of an automated discovery process that scans for and registers all command implementations.
- **Scope:** The object instance is long-lived, persisting for the entire server session. It is stored within the CommandManager's internal registry.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only upon server shutdown, when the CommandManager clears its registry.

## Internal State & Concurrency
- **State:** This class is effectively **immutable** after its constructor completes. The list of sub-commands is populated during instantiation and is not modified during the server's runtime. It holds no mutable game state.
- **Thread Safety:** Command execution is managed and serialized by the server's core engine, typically on a dedicated thread from the main game loop. As such, instances of this class are not subject to concurrent modification from multiple threads and are considered **conditionally thread-safe** within the context of the engine's execution model. Direct, multi-threaded access is an unsupported and dangerous pattern.

## API Surface
The public contract of this class is primarily for consumption by the command registration system. It is not intended for direct invocation by other game systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HitboxCollisionCommand() | Constructor | O(1) | Instantiates the command group and registers its child sub-commands. This is the sole entry point for its lifecycle. |

## Integration Patterns

### Standard Usage
This class is not designed for programmatic use. It is automatically discovered and managed by the server's command system. A developer or server administrator interacts with it exclusively through the server console or in-game chat.

```
// This class is not invoked directly in code.
// Usage is performed via the server console.

// Example: Add a new debug hitbox
> /hitboxcollision add <entityId> <type>

// Example: Remove a debug hitbox
> /hitboxcollision remove <entityId>
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new HitboxCollisionCommand()` outside of a dedicated command registration system. An un-registered instance is inert and will not be recognized by the server.
- **Programmatic Execution:** Do not attempt to acquire an instance of this class to call its methods directly. All command execution must be routed through the central CommandManager to ensure correct parsing, permission checks, and state management.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a potential modification of game state by one of its sub-commands. This class acts as a routing step in that pipeline.

> Flow:
> Server Console Input -> Command Parser -> **HitboxCollisionCommand** (Routing) -> Sub-Command Executor (e.g., HitboxCollisionAddCommand) -> Game World State Change


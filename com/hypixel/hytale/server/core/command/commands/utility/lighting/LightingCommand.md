---
description: Architectural reference for LightingCommand
---

# LightingCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.lighting
**Type:** Singleton

## Definition
```java
// Signature
public class LightingCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The LightingCommand class serves as a top-level namespace and dispatcher for all lighting-related server commands. It is a concrete implementation of the Composite design pattern, acting as a non-terminal node in the command tree. Its sole responsibility is to group related sub-commands—such as LightingCalculationCommand and LightingInvalidateCommand—under a single, user-friendly entry point: *lighting*.

This class does not contain any logic related to the game's lighting engine. Instead, it functions as a routing mechanism within the server's Command System. When a user executes a command beginning with *lighting*, the server's central CommandManager dispatches the request to this object. The parent class, AbstractCommandCollection, then takes responsibility for parsing the subsequent arguments and delegating execution to the appropriate registered sub-command.

This architectural pattern simplifies command registration and promotes modularity by decoupling the parent command from the implementation details of its children.

## Lifecycle & Ownership
- **Creation:** A single instance of LightingCommand is instantiated by the server's central CommandManager during the server bootstrap phase. This occurs when the system scans for and registers all available commands.
- **Scope:** The object's lifecycle is tied directly to the server session. It is created once upon server start and persists in the CommandManager's registry until server shutdown.
- **Destruction:** The instance is eligible for garbage collection only when the CommandManager is cleared during the server shutdown sequence.

## Internal State & Concurrency
- **State:** The LightingCommand is effectively stateless and immutable after construction. Its internal state consists of a collection of sub-commands, which is populated exclusively within the constructor and is not modified thereafter.
- **Thread Safety:** This class is inherently thread-safe. As its internal collection of sub-commands is treated as read-only after initialization, multiple threads can safely access it without synchronization. **Warning:** While this container is thread-safe, the sub-commands it holds may have their own state and concurrency constraints that must be respected.

## API Surface
The public API is almost entirely inherited from AbstractCommandCollection. The constructor is the only significant public member defined directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LightingCommand() | Constructor | O(1) | Initializes the command group. Registers the primary name "lighting", an alias "light", and all associated lighting sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked implicitly by the server's command processing system when a user or administrator types the command into the server console.

```java
// This class is not invoked programmatically.
// It is triggered by console input.

// Example Console Commands:
> lighting invalidate
> lighting info
> light send
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class using *new LightingCommand()*. The server's CommandManager is solely responsible for the lifecycle of all command objects. Manual instantiation will result in a non-functional command that is not registered with the system.
- **Runtime Modification:** Do not attempt to add or remove sub-commands from this collection after its initial construction. The internal state is not designed for modification at runtime and doing so can lead to race conditions and unpredictable behavior.

## Data Pipeline
LightingCommand acts as a control-flow component, not a data-processing one. Its role in the pipeline is to route requests.

> Flow:
> Console Input -> Server Network Layer -> CommandManager Parser -> **LightingCommand** -> Sub-Command (e.g., LightingInvalidateCommand) -> Lighting Engine -> Console Output


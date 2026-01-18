---
description: Architectural reference for PlayerCameraDemoSubCommand
---

# PlayerCameraDemoSubCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.camera
**Type:** Transient

## Definition
```java
// Signature
public class PlayerCameraDemoSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PlayerCameraDemoSubCommand class is a structural component within the server's command processing system. It functions as a **command collection**, a non-executable node in the command tree that groups related sub-commands under a common namespace.

Its primary role is to act as a router. When the command system parses an input like `/camera demo ...`, this class is responsible for delegating the remainder of the command string to one of its registered children, such as PlayerCameraDemoActivateCommand or PlayerCameraDemoDeactivateCommand. It provides no game logic itself; it is purely for organization and discoverability of related camera demonstration functionalities.

This pattern of nesting AbstractCommandCollection instances allows for the creation of a deep, hierarchical command structure, mirroring a standard command-line interface (CLI) design.

### Lifecycle & Ownership
- **Creation:** Instantiated by its parent, likely a `PlayerCameraCommand`, during the server's bootstrap phase where all commands are registered with the central CommandManager.
- **Scope:** The object's lifetime is tied to the server session. It is created once upon server start and persists in the command registry's memory until server shutdown.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the CommandManager is cleared, typically during a server shutdown sequence.

## Internal State & Concurrency
- **State:** This class is effectively **immutable** after construction. The internal list of sub-commands is populated within the constructor and is not designed to be modified thereafter. It holds no mutable game state.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless and immutable nature. However, command execution is managed by the server's core command processing loop, which imposes its own threading model. It is unsafe to assume that the *sub-commands* it contains can be executed concurrently. All command logic should be considered to execute on a designated game thread unless explicitly documented otherwise.

## API Surface
The public contract is almost entirely defined by its constructor, which establishes its identity and composition within the command tree.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerCameraDemoSubCommand() | Constructor | O(1) | Initializes the command collection with the name "demo" and registers its child commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be instantiated and registered by a parent command collection to build out the command hierarchy.

```java
// Within a parent command's constructor (e.g., PlayerCameraCommand)
// This is the correct pattern for integrating this component.
public class PlayerCameraCommand extends AbstractCommandCollection {
    public PlayerCameraCommand() {
        super("camera", "...");
        this.addSubCommand(new PlayerCameraDemoSubCommand()); // Correct usage
        // ... other sub-commands
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation without Registration:** Creating an instance of PlayerCameraDemoSubCommand without adding it to a parent command via `addSubCommand` will result in an orphaned object that is never registered with the command system and will be immediately garbage collected. It will not be executable.
- **Stateful Logic:** Do not add fields or mutable state to this class. A command collection should be a stateless router. Any logic or state should be implemented within the final, executable command classes (e.g., PlayerCameraDemoActivateCommand).

## Data Pipeline
This class acts as a routing node in a control flow, not a data processing pipeline. It directs the flow of command execution based on user input.

> Flow:
> User Input (`/camera demo activate`) -> Command Parser -> Parent Command (`PlayerCameraCommand`) -> **`PlayerCameraDemoSubCommand`** -> Child Command (`PlayerCameraDemoActivateCommand`) -> Command Executor -> Game World Mutation


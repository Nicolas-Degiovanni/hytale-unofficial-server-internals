---
description: Architectural reference for OpCommand
---

# OpCommand

**Package:** com.hypixel.hytale.server.core.permissions.commands.op
**Type:** Transient

## Definition
```java
// Signature
public class OpCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The OpCommand class is a structural component within the server's command processing system. It does not implement any command logic itself. Instead, it functions as a **Command Collection**, a container that groups related sub-commands under a single, top-level namespace: *op*.

This class is an implementation of the **Composite Pattern**. It acts as a non-terminal node in the command tree, delegating execution to its children (OpSelfCommand, OpAddCommand, OpRemoveCommand), which are the terminal leaf nodes that contain the actual logic. Its primary architectural role is to organize the command structure, making the operator permission commands accessible via a unified and intuitive entry point like `/op add <player>` or `/op remove <player>`.

The override of `canGeneratePermission` to return false is a critical design choice. It signals to the command system that there is no single permission node for the entire `/op` command group (e.g., `hytale.command.op`). Instead, permissions are managed granularly by the individual sub-commands, allowing for more fine-grained access control.

## Lifecycle & Ownership
- **Creation:** OpCommand is instantiated once by the server's central command registry during the server bootstrap or plugin loading phase. The system discovers command classes and creates instances to build its dispatch table.
- **Scope:** This object is session-scoped. It persists in memory for the entire duration of the server's runtime, held as a reference within the command registry.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down or the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless and effectively immutable**. Its internal state, the list of sub-commands, is populated exclusively within the constructor and is not modified during its lifecycle. It holds no runtime data or caches.
- **Thread Safety:** OpCommand is inherently **thread-safe**. As a stateless container, it can be safely accessed by the command dispatcher from any thread without synchronization. All concurrency management is the responsibility of the underlying command execution engine and the concrete sub-command implementations.

## API Surface
The public API is minimal and intended for framework use only. Direct interaction by developers is not expected.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OpCommand() | Constructor | O(1) | Initializes the command collection, setting its name and registering its child sub-commands. |
| canGeneratePermission() | boolean | O(1) | Overrides parent behavior to prevent automatic permission node generation for the entire collection. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is automatically discovered and registered by the server's command system. A server administrator or a user with sufficient permissions interacts with it via the server console or in-game chat.

```text
// Example user interaction via the server console

// Grants operator status to the player named "Notch"
> op add Notch

// Removes operator status from the player named "Steve"
> op remove Steve
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance of OpCommand using `new OpCommand()`. An un-registered command object is inert and serves no purpose, as the server's command dispatcher will be unaware of it.
- **Adding Logic:** Do not extend this class to add execution logic. Its purpose is strictly for composition. All command logic must be implemented in the leaf sub-commands (e.g., OpAddCommand).
- **Dynamic Modification:** Do not attempt to add or remove sub-commands after construction. The command hierarchy is designed to be static and defined at startup.

## Data Pipeline
OpCommand acts as a routing point in the data flow of command processing. It receives control from the main dispatcher and forwards it to the appropriate sub-command based on the input arguments.

> Flow:
> User Input (`/op add Player`) -> Server Command Parser -> Command Dispatcher (matches "op") -> **OpCommand** -> Sub-Command Router (matches "add") -> OpAddCommand -> Execution & Feedback


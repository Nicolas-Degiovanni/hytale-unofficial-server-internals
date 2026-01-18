---
description: Architectural reference for MountCheckCommand
---

# MountCheckCommand

**Package:** com.hypixel.hytale.builtin.mounts.commands
**Type:** Transient

## Definition
```java
// Signature
public class MountCheckCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The MountCheckCommand is a server-side command handler responsible for inspecting and reporting the mount status of a target player. It functions as a leaf node within the server's hierarchical command system, specifically as a subcommand for a parent "mount" command.

By extending AbstractTargetPlayerCommand, this class delegates the complex logic of argument parsing, player lookup, and permission validation to its parent. This allows MountCheckCommand to focus exclusively on its core responsibility: querying the Entity Component System (ECS) for the presence and state of a MountedComponent on a given entity.

This class serves as a diagnostic and administrative tool, providing a direct interface for server operators to debug entity states without requiring more complex tooling. It is a pure read-only operator; it never mutates game state.

## Lifecycle & Ownership
- **Creation:** A single instance of MountCheckCommand is created by the command registration system during server bootstrap. It is discovered and instantiated alongside other commands within the "mounts" builtin module.
- **Scope:** The object instance is effectively a singleton managed by the server's central command registry. It persists for the entire lifetime of the server.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down or if the parent "mounts" module is dynamically unloaded.

## Internal State & Concurrency
- **State:** MountCheckCommand is **stateless**. It maintains no instance fields that store data between invocations. All required information, such as the command source, target player, and world context, is passed as method-local arguments into the execute method. The only fields are static final Message objects, which are immutable constants.
- **Thread Safety:** This class is not designed for concurrent execution. The command system guarantees that the execute method is invoked from a single, predictable thread, typically the main server thread. Direct invocation from other threads is unsupported and will lead to race conditions when accessing the World or Store arguments.

## API Surface
The primary contract is the protected execute method, which is invoked by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Fetches the MountedComponent for the target entity and sends a localized status message to the command issuer. This is a read-only operation. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be registered with the server's command system and invoked by an authorized user (e.g., a server administrator) via the chat console.

The framework handles the entire invocation lifecycle:
1. A user types a command like `/mount check PlayerName`.
2. The command system parses the input and identifies MountCheckCommand as the handler.
3. The framework resolves "PlayerName" to a valid entity reference.
4. The framework populates all arguments and invokes the `execute` method.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new MountCheckCommand()`. The command system is responsible for the object's lifecycle. Manually creating an instance will result in a non-functional object that is not registered to handle any user input.
- **Programmatic Invocation:** Avoid calling the `execute` method directly from other game systems. Doing so bypasses critical framework features like permission checks and argument validation. If you need to query an entity's mount status from code, you should directly query the `Store` for the `MountedComponent` instead of routing through the command system.

## Data Pipeline
MountCheckCommand acts as a terminal endpoint in a user-initiated data query flow. It does not process a continuous stream of data; rather, it handles discrete requests.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Parser -> **MountCheckCommand.execute** -> Store (Component Read) -> PlayerRef (Message Send) -> Server Network Layer -> Client Chat UI


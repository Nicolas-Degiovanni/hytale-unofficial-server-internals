---
description: Architectural reference for PlayerEffectSubCommand
---

# PlayerEffectSubCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.effect
**Type:** Transient

## Definition
```java
// Signature
public class PlayerEffectSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PlayerEffectSubCommand class is a **Command Group** within the server's command processing system. It does not implement any executable logic itself. Instead, its sole architectural purpose is to act as a structural container and routing node for a collection of related subcommands.

By extending AbstractCommandCollection, it inherits the responsibility of managing a hierarchy of commands. In this case, it groups the *apply* and *clear* operations for player effects under a single, user-facing command: **effect**.

When a user executes a command like `/effect apply ...`, the server's central CommandManager first dispatches the request to this PlayerEffectSubCommand object based on the "effect" token. The parent AbstractCommandCollection logic then parses the subsequent token ("apply") and forwards the execution context to the corresponding registered child, PlayerEffectApplyCommand.

This pattern is fundamental to creating a clean, hierarchical, and extensible command-line interface for server administrators and players.

### Lifecycle & Ownership
-   **Creation:** A single instance is created during the server's bootstrap phase by a central command registration service, likely a CommandManager or a similar bootstrap coordinator.
-   **Scope:** The object instance is held in memory by the command registration service for the entire lifetime of the server. It acts as a singleton in practice, though it is not enforced by the class design itself.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down or if a hot-reload mechanism unregisters the command group.

## Internal State & Concurrency
-   **State:** The state of this object is effectively **immutable** after construction. The list of subcommands is populated within the constructor and is not designed to be modified at runtime. All state, such as the command name and the collection of children, is configured once and then treated as read-only.

-   **Thread Safety:** This class is **conditionally thread-safe**. The internal state is not protected by locks, as it is not expected to be mutated after its initial creation on the server's main thread. Command execution, which involves reading from its subcommand list, is managed and synchronized by the server's central command processing loop. Direct, concurrent modification from other threads would be an architectural violation and lead to undefined behavior.

## API Surface
The public API is minimal, as the primary interaction is through the constructor during system initialization. All other behavior is inherited from AbstractCommandCollection and is invoked internally by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerEffectSubCommand() | Constructor | O(k) | Initializes the command group with the name "effect" and registers its *k* child commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation in game logic. Its integration is declarative, part of the server's command definition process. A central registry is responsible for its instantiation.

```java
// Example from a hypothetical CommandManager service
// This code runs once at server startup.

CommandRegistry registry = serverContext.getCommandRegistry();

// The PlayerEffectSubCommand is instantiated and registered as a top-level command.
registry.registerCommand(new PlayerEffectSubCommand());
```

### Anti-Patterns (Do NOT do this)
-   **Adding Business Logic:** Do not add methods or fields containing game logic to this class. It is a container. All logic must be delegated to its leaf-node subcommands, such as PlayerEffectApplyCommand.
-   **Dynamic Modification:** Do not attempt to call `addSubCommand` or otherwise modify the command collection after its initial construction. The command system treats its registrations as immutable during runtime.
-   **Manual Invocation:** Never instantiate and call methods on this class to try and execute a command. Command execution must flow through the server's central command dispatcher to ensure permissions, context, and argument parsing are handled correctly.

## Data Pipeline
The PlayerEffectSubCommand acts as a routing step in the server's command processing pipeline. It receives the execution context and delegates it to a more specific handler.

> Flow:
> Player Chat Input (`/effect clear @s`) -> Network Packet -> Server Command Parser -> Command Registry Dispatch -> **PlayerEffectSubCommand** (Routes based on "clear") -> PlayerEffectClearCommand (Executes logic) -> Player State Mutation


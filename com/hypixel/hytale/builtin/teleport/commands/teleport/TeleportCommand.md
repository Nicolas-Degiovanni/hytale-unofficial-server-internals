---
description: Architectural reference for TeleportCommand
---

# TeleportCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Transient

## Definition
```java
// Signature
public class TeleportCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The TeleportCommand class is a **Command Aggregator**. It does not implement any teleportation logic itself. Instead, its sole responsibility is to define the command structure for the primary `/tp` and `/teleport` commands and to act as a dispatcher, delegating execution to more specialized command handlers.

This class functions as the root node in a command tree for all teleport-related actions. It leverages the **Composite Pattern** by treating a collection of distinct command objects—its *variants* and *sub-commands*—as a single, unified interface from the perspective of the server's command registry.

-   **Usage Variants:** These define different argument signatures for the base `/tp` command. For example, `tp <player>` is handled by TeleportToPlayerCommand, while `tp <x> <y> <z>` is handled by TeleportToCoordinatesCommand. The system selects the correct variant based on the provided arguments.
-   **Sub-Commands:** These are distinct commands nested under the `/tp` namespace, such as `/tp home` or `/tp all`. Each is a self-contained command object responsible for its own logic.

By centralizing the command definitions here, the system maintains a clean and extensible structure for a feature with many potential variations.

## Lifecycle & Ownership
-   **Creation:** A single instance of TeleportCommand is created by the server's core CommandRegistry during the server bootstrap phase. The registry scans for command definitions and instantiates them to build the server's command graph.
-   **Scope:** The instance persists for the entire server session. It is part of the server's static command definition and does not hold any per-player or per-world state.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency
-   **State:** This object is effectively **immutable** after its constructor completes. The constructor populates internal collections of variants and sub-commands, which are not modified during runtime. It is a stateless definition object.
-   **Thread Safety:** This class is inherently **thread-safe**. Because its internal state is static after initialization, it can be safely accessed by multiple server threads processing commands from different players concurrently without requiring locks or synchronization. The parent AbstractCommandCollection is designed for this concurrent access pattern.

## API Surface
The public contract of this class is fulfilled entirely within its constructor, which configures the command's properties and registers its children. It has no other public methods intended for external use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TeleportCommand() | Constructor | O(1) | Configures the root `tp` command. Sets its primary name, description key, permission group, and alias. Registers all child usage variants and sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended to be instantiated or used directly in game logic or by other systems. It is exclusively used by the server's command registration system. Developers seeking to add new commands should follow this pattern by creating their own class extending AbstractCommand or AbstractCommandCollection.

```java
// Example of a plugin registering a similar command collection
// This code would exist within a server bootstrap or plugin entry point.

CommandRegistry registry = server.getCommandRegistry();

// The system discovers and instantiates commands automatically,
// but manual registration would look like this:
registry.register(new TeleportCommand());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new TeleportCommand()` in your game logic. The object has no purpose outside of its registration with the CommandRegistry. Creating an instance will have no effect on the server's available commands.
-   **State Modification:** Do not attempt to modify the command's state after construction (e.g., via reflection). The command graph is assumed to be static once the server is running. Dynamically adding or removing sub-commands at runtime is unsupported and will lead to unpredictable behavior.

## Data Pipeline
TeleportCommand acts as a routing and delegation point in the command processing pipeline. It receives parsed input and directs it to the appropriate handler.

> Flow:
> Player Chat Input (`/tp PlayerA PlayerB`) → Server Network Listener → Command Parser → CommandRegistry (resolves "tp" to TeleportCommand) → **TeleportCommand** (matches arguments to TeleportOtherToPlayerCommand variant) → Command Executor (invokes the variant) → World State Update → Network Packet (sends position update to clients)


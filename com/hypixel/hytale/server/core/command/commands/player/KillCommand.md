---
description: Architectural reference for KillCommand
---

# KillCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Command Handler

## Definition
```java
// Signature
public class KillCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The KillCommand class is a server-side command handler responsible for implementing the `/kill` chat command. It is a concrete implementation within the server's broader Command System, which is responsible for parsing, validating, and dispatching user-entered commands.

Architecturally, this class demonstrates two key engine patterns:

1.  **Command Variants:** The primary class handles the self-targeting variant (`/kill`), while the nested static class `KillOtherCommand` handles the variant for targeting other players (`/kill <player>`). This composition pattern keeps related command logic encapsulated within a single file while separating concerns.
2.  **Decoupled Damage System:** Crucially, this command does not directly manipulate a player's health value. Instead, it integrates with the server's Entity-Component-System (ECS) by creating a `Damage` object and applying a `DeathComponent` to the target entity. This ensures that dying from a command follows the exact same game logic and event pipeline as dying from any other source, such as mob attacks or environmental hazards. This approach guarantees that all associated systems, like death messages, respawn logic, and potential event listeners, are triggered correctly.

The class operates on `PlayerRef` and `EntityStore` objects, the fundamental handles for interacting with entity state within the ECS framework.

### Lifecycle & Ownership
-   **Creation:** A single instance of KillCommand is created by the server's `CommandRegistry` or a similar service during the server bootstrap phase. The system scans for classes that implement command interfaces and instantiates them for registration.
-   **Scope:** The KillCommand object is a long-lived singleton for the duration of the server's runtime. It is registered once and persists until server shutdown. The *execution* of the command, however, is transient and scoped to a single invocation.
-   **Destruction:** The object is dereferenced and eligible for garbage collection when the `CommandRegistry` is cleared during server shutdown.

## Internal State & Concurrency
-   **State:** The KillCommand class is effectively stateless and immutable after construction. Its properties, such as the command name and required permissions, are configured in the constructor and do not change. All state required for execution (e.g., the command sender, the target player) is passed into the `execute` methods via the `CommandContext`.

-   **Thread Safety:** This class is designed to be thread-safe through explicit thread management. The `KillOtherCommand` variant receives a command invocation, which may originate from a network or general-purpose thread. It then uses `world.execute` to schedule the entity modification logic onto the specific thread responsible for managing that entity's world. This is a critical safety mechanism that prevents race conditions and ensures all modifications to the ECS `EntityStore` are serialized and processed during the world's tick cycle.

    **WARNING:** Directly modifying entity components from the `executeSync` method without scheduling the operation on the appropriate world thread would corrupt game state.

## API Surface
The public contract is primarily for the Command System framework, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| KillCommand() | constructor | O(1) | Constructs the command, defines its name, description, permissions, and registers the `KillOtherCommand` variant. |
| execute(...) | protected void | O(1) | Framework-invoked method. Applies a fatal `DeathComponent` to the player who executed the command. |
| KillOtherCommand.executeSync(...) | protected void | O(1) | Framework-invoked method. Parses the target player argument and schedules a fatal `DeathComponent` to be applied to them on their world's thread. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked automatically by the server's command handler when a player with the appropriate permissions types the command into chat.

**In-Game Usage:**
```
/kill
/kill Notch
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new KillCommand()` in your own code. The server's `CommandRegistry` is the sole owner and manager of command instances.
-   **Direct Invocation:** Never call the `execute` or `executeSync` methods directly. Doing so bypasses the entire command processing pipeline, including critical steps like permission validation, argument parsing, and context creation.
-   **Unsafe World Modification:** When creating similar commands, do not modify entity state from a command's execution method without scheduling the work on the entity's world thread, as demonstrated by `world.execute`.

## Data Pipeline
The flow for executing `/kill <player>` demonstrates the interaction between the network, command, and game logic layers.

> Flow:
> Player Chat Input -> Server Network Packet -> Command Parser -> **CommandContext** is built -> `CommandRegistry` dispatches to `KillCommand.KillOtherCommand` -> `executeSync` validates target and schedules task -> **World Thread** executes task -> `DeathComponent.tryAddComponent` modifies the **EntityStore** -> Death & Respawn Systems are notified -> Player receives death message.


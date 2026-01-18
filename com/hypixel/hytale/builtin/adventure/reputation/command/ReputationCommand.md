---
description: Architectural reference for ReputationCommand
---

# ReputationCommand

**Package:** com.hypixel.hytale.builtin.adventure.reputation.command
**Type:** Transient

## Definition
```java
// Signature
public class ReputationCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The **ReputationCommand** class serves as a structural root for all reputation-related server commands. It does not implement any game logic itself. Instead, it functions as a command router or a collection within the server's broader command system, which is managed by the **CommandManager**.

By extending **AbstractCommandCollection**, it inherits the responsibility of grouping related sub-commands under a single, user-facing entry point—in this case, the `/reputation` command. Its sole architectural purpose is to aggregate and register its child commands (**ReputationAddCommand**, **ReputationSetCommand**, etc.) with the command dispatcher. When a player or administrator executes a command like `/reputation add ...`, the command system first routes the request to this parent object, which then delegates execution to the appropriate sub-command based on the subsequent arguments.

This pattern simplifies command registration and promotes a clean, hierarchical command structure that is easy for users to discover and for developers to extend.

### Lifecycle & Ownership
- **Creation:** An instance of **ReputationCommand** is created once during the server's bootstrap sequence. The central **CommandManager** or a related service locator is responsible for discovering and instantiating all built-in command classes, including this one.
- **Scope:** The object instance is a long-lived singleton managed by the command system. It persists for the entire duration of the server session.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the **CommandManager** clears its command registry.

## Internal State & Concurrency
- **State:** This class is stateless. Its internal list of sub-commands is populated exclusively within the constructor and is not modified thereafter, making the object effectively immutable after initialization. All stateful operations are delegated to the sub-commands it contains.
- **Thread Safety:** The **ReputationCommand** object is inherently thread-safe due to its immutable nature. However, the execution of the commands it delegates to is managed by the server's command execution engine. This engine is responsible for ensuring that command logic is executed on the correct thread, typically the main server thread, to prevent race conditions when modifying game state.

## API Surface
The programmatic API surface of this class is minimal and intended only for initialization by the command system. The primary interface is the command-line behavior it exposes to server administrators.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ReputationCommand() | Constructor | O(1) | Instantiates the command collection and registers its four child sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic or plugin developers. It is automatically discovered and registered by the server's core command system at startup. The standard interaction is performed by a server administrator via the console.

**Administrator Interaction Example:**
```shell
# This command is routed through ReputationCommand to ReputationAddCommand
/reputation add player_name faction_id 100
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually using `new ReputationCommand()`. An un-registered command object is inert and serves no purpose, consuming memory without providing any functionality. The server's **CommandManager** handles the entire lifecycle.
- **Stateful Modification:** Do not attempt to extend this class to add mutable state. A command collection should remain a stateless router. All stateful logic must be encapsulated within the terminal sub-commands that perform the final action.

## Data Pipeline
The **ReputationCommand** acts as an early routing stage in the server's command processing pipeline. It receives a parsed command structure and dispatches it to a more specific handler.

> Flow:
> Console Input -> Network Packet -> Server Command Parser -> Command Dispatcher -> **ReputationCommand** (Router) -> ReputationAddCommand (Executor) -> Reputation Service -> Game State Update


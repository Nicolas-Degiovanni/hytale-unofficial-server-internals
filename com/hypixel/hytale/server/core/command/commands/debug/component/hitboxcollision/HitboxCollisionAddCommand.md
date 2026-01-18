---
description: Architectural reference for HitboxCollisionAddCommand
---

# HitboxCollisionAddCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.component.hitboxcollision
**Type:** Utility

## Definition
```java
// Signature
public class HitboxCollisionAddCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The HitboxCollisionAddCommand class is a component of the server-side Command System. It does not implement any command logic itself; instead, it functions as a *command collection* or a routing namespace for a group of related sub-commands. Its sole architectural purpose is to group the `add entity` and `add self` functionalities under the common `add` verb within the larger `/hitboxcollision` command structure.

This class acts as a bridge between the user-facing command-line interface and direct manipulation of the server's Entity-Component-System (ECS). By parsing user input, it provides a controlled mechanism to dynamically attach a new HitboxCollision component to an existing entity, thereby altering its physical behavior at runtime.

The core logic is delegated to its two inner classes:
*   **HitboxCollisionAddEntityCommand:** Targets any arbitrary entity within the world.
*   **HitboxCollisionAddSelfCommand:** A convenience command that targets the entity associated with the command's executor (typically the player).

This pattern of a collection class with nested command implementations is standard within the engine for creating hierarchical and organized command trees.

### Lifecycle & Ownership
- **Creation:** A single instance of HitboxCollisionAddCommand is created by the server's central CommandRegistry during the server bootstrap phase. The system scans for classes that extend command abstractions and instantiates them.
- **Scope:** The object is a stateless container that persists for the entire lifetime of the server. It is effectively a singleton managed by the command framework.
- **Destruction:** The instance is garbage collected only upon server shutdown when the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class and its nested sub-commands are fundamentally stateless. They hold immutable configuration for command arguments, but they do not store any data related to a specific execution. All state required for execution, such as the target entity and world reference, is provided via the CommandContext at the moment of invocation.

- **Thread Safety:** **This class is not thread-safe.** Command execution involves direct mutation of the ECS EntityStore, which is a non-concurrent data structure. The command system guarantees that all command execution is serialized and occurs exclusively on the main server thread, which also manages the world tick.

    **WARNING:** Any attempt to invoke command logic from an asynchronous task or a different thread will lead to world state corruption, race conditions, and likely server crashes.

## API Surface
The primary programmatic contract is with the command framework, which invokes the protected `execute` methods on the inner classes. The user-facing contract is the command syntax itself.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HitboxCollisionAddEntityCommand | Sub-Command | O(log N) | Adds a HitboxCollision component to a specified entity. Complexity is tied to the ECS store's component addition operation. |
| HitboxCollisionAddSelfCommand | Sub-Command | O(log N) | Adds a HitboxCollision component to the command issuer's entity. Complexity is tied to the ECS store's component addition operation. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. Its functionality is exposed exclusively through the server console or in-game chat interface.

The standard interaction pattern is to execute the command as a server administrator or a user with appropriate permissions.

**Command-Line Syntax:**
```sh
# Add a hitbox to a specific entity by its ID
/hitboxcollision add entity <entityId> <hitboxCollisionConfig>

# Add a hitbox to your own player entity
/hitboxcollision add self <hitboxCollisionConfig>
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HitboxCollisionAddCommand()` in game logic. The command system manages the lifecycle of command objects. A manually created instance will not be registered and will serve no purpose.
- **Manual Execution:** Avoid calling the `execute` method directly. This bypasses critical framework services, including argument parsing, permission validation, and context creation, which will result in unpredictable behavior and exceptions.
- **Stateful Logic:** Do not modify the command classes to hold state between executions. Command objects must remain stateless to ensure predictable behavior across the server.

## Data Pipeline
The flow of data from user input to world modification is managed entirely by the command framework. This class represents a single step in that pipeline where the final, parsed data is used to perform a state change.

> Flow:
> User Input String (`/hitboxcollision add...`) -> Server Command Dispatcher -> **HitboxCollisionAddCommand** (Router) -> **HitboxCollisionAddEntityCommand** (Executor) -> Argument Parsers (`ArgTypes`) -> `execute` method invocation -> `Store.addComponent` call -> World State Mutation -> Feedback `Message` -> User



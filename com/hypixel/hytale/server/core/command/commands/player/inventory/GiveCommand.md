---
description: Architectural reference for GiveCommand
---

# GiveCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.inventory
**Type:** Registered Command Handler

## Definition
```java
// Signature
public class GiveCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The GiveCommand class is a server-side command handler responsible for the in-game *give* command. It acts as the primary bridge between parsed player chat input and the server's core inventory and entity systems.

Architecturally, this class embodies the Command pattern. It encapsulates all information needed to perform the action of adding an item to a player's inventory, including argument parsing, permission validation, and execution logic.

A key design feature is its use of a nested private class, GiveOtherCommand, to implement a "usage variant". The top-level GiveCommand handles giving an item to the command's executor, while the nested class handles the more privileged action of giving an item to another player. This pattern cleanly separates permission requirements and execution logic for related but distinct functionalities under a single command name.

The class leverages the engine's declarative argument system (e.g., withRequiredArg, withDefaultArg) to define its syntax and automatically handle type conversion and validation of user input.

## Lifecycle & Ownership
- **Creation:** A single instance of GiveCommand is created by the server's command registration system during the server bootstrap sequence. It is not instantiated on a per-request basis.
- **Scope:** The instance persists for the entire lifetime of the server. As a stateless handler, one instance is sufficient to process all incoming *give* commands.
- **Destruction:** The object is dereferenced and eligible for garbage collection only upon server shutdown when the central command registry is cleared.

## Internal State & Concurrency
- **State:** The GiveCommand instance itself is effectively stateless and immutable after construction. Its fields are final references to argument definitions. The command's execution, however, manipulates the highly mutable state of a Player entity's inventory component.

- **Thread Safety:** This class demonstrates a critical concurrency pattern for server-side entity modification.
    - The primary `execute` method, inherited from AbstractPlayerCommand, is invoked on the specific World thread that owns the executing player. This guarantees thread-safe access to that player's components.
    - The nested GiveOtherCommand's `executeSync` method is called from a generic command processing thread. It **must not** modify the target player's state directly. Instead, it correctly schedules the inventory modification logic onto the target player's World thread using `world.execute`. This is the mandatory pattern for cross-thread entity interaction to prevent race conditions and data corruption.

## API Surface
The public contract of this class is not a set of methods for developers to call, but rather the command syntax it registers with the server.

| Command Variant | Permission | Description |
| :--- | :--- | :--- |
| `/give <item> [quantity] [metadata]` | `hytale.command.give.self` | Gives the specified item to the player executing the command. Quantity defaults to 1. Metadata is an optional BSON string. |
| `/give <player> <item> [quantity] [metadata]` | `hytale.command.give.other` | Gives the specified item to the target player. Requires a higher permission level. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked by the server's command processor in response to player input.

```text
// Player enters the following into the in-game chat
/give hytale:stone_sword 1
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new GiveCommand()`. The command system handles instantiation and registration. A manually created instance will not be registered and will have no effect.
- **Direct Method Invocation:** Do not call the `execute` or `executeSync` methods directly. Doing so bypasses the entire command processing pipeline, including critical permission checks, argument parsing, and thread safety guarantees.
- **Cross-Thread State Modification:** When interacting with entities, never modify their components from an arbitrary thread. Always follow the pattern shown in GiveOtherCommand by scheduling work on the entity's owner World thread via `world.execute`.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a change in world state and a feedback message to the user.

> Flow:
> Player Chat Input -> Network Packet -> Command Parser -> **GiveCommand** -> Argument Validation -> World Thread Scheduler -> Player Inventory Component -> ItemStackTransaction -> Feedback Message Generation -> Network Packet -> Player Chat Display


---
description: Architectural reference for LeaveCommand
---

# LeaveCommand

**Package:** com.hypixel.hytale.builtin.portals.commands.player
**Type:** Transient

## Definition
```java
// Signature
public class LeaveCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The LeaveCommand class is a concrete implementation of the Command Pattern, designed to handle player-initiated exits from special world instances, specifically "Portal Worlds". It serves as a user-facing entry point, translating a simple chat command, `/leave`, into a complex server-side operation.

Architecturally, this class acts as a controller that bridges player input with the core instance management system. Its primary responsibility is not to contain the logic for leaving an instance, but to orchestrate the process by:
1.  **Validating Context:** It first verifies that the player is inside a valid Portal World by querying for the presence of a `PortalWorld` resource in the current world's `Store`. This makes the command context-sensitive and prevents its execution in inappropriate environments.
2.  **Executing Pre-Exit Hooks:** Before triggering the world transition, it performs a gameplay-specific side effect: cleansing the player's inventory of "cursed" items via the `CursedItems` utility. This demonstrates a pattern of commands handling both system-level operations and related gameplay mechanics.
3.  **Delegating to a Subsystem:** The core logic of exiting the instance is delegated to the `InstancesPlugin`. This is a critical design choice, decoupling the command interface from the underlying implementation of instance management.

## Lifecycle & Ownership
-   **Creation:** An instance of LeaveCommand is created by the server's command registration system during server startup or plugin loading. It is discovered and registered as the handler for the "leave" command string.
-   **Scope:** The LeaveCommand object persists for the entire server session, held as a reference within the central command dispatcher or registry. It is effectively a singleton from the perspective of the command system.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server shuts down or the parent plugin is unloaded, at which point the command is removed from the registry.

## Internal State & Concurrency
-   **State:** This class is stateless. All data required for its operation is provided through the parameters of the `execute` method, such as the `CommandContext`, `Store`, and `PlayerRef`. The static `Message` fields are immutable and defined at compile time. This stateless design is ideal for command handlers, ensuring predictable and isolated executions.
-   **Thread Safety:** The `execute` method is invoked by the server's command processing system, which operates within the main server thread or a world-specific thread. The class is not designed for concurrent invocation and relies on the engine's single-threaded execution model per world tick to guarantee safety when mutating the player's state via the provided `Store` and `Ref`.

## API Surface
The public contract is defined by its constructor and the overridden `execute` method, which is the entry point for the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LeaveCommand() | constructor | O(1) | Instantiates the command and registers its name and description with the superclass. |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the leave logic. Complexity is O(N) where N is the number of items in the player's inventory, plus the complexity of the delegated `InstancesPlugin.exitInstance` call. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. The system is designed for a player to trigger its execution by typing the command into the game's chat console.

> **Player Action:**
> `/leave`

The server's command parser identifies the "leave" keyword, finds the registered LeaveCommand instance, and invokes its `execute` method, passing in the full context of the player and the world they currently inhabit.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new LeaveCommand()` in your own code. The command system handles instantiation and registration. Creating a new instance will result in an object that is not registered to handle any chat commands.
-   **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the server's command permission checks, context validation, and dispatching logic. The correct way to make a player leave an instance programmatically is to call the underlying subsystem directly, for example, `InstancesPlugin.exitInstance(ref, store)`.

## Data Pipeline
The flow of data and control for this command begins with the player and terminates with a world state change, mediated by several server systems.

> Flow:
> Player Chat Input (`/leave`) -> Server Network Handler -> Command Parser -> Command Dispatcher -> **LeaveCommand.execute()**
> 1.  Reads `PortalWorld` resource from `Store`.
> 2.  Reads `Player` component from `Store` to access inventory.
> 3.  Delegates to `CursedItems.uncurseAll()`.
> 4.  Delegates to `InstancesPlugin.exitInstance()`.
>
> -> Instance Management System -> Player Teleportation & World Transition Logic


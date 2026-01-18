---
description: Architectural reference for InventoryClearCommand
---

# InventoryClearCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.inventory
**Type:** Transient

## Definition
```java
// Signature
public class InventoryClearCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The InventoryClearCommand is a concrete implementation of the Command Pattern, designed to integrate with the server's central command processing system. It encapsulates a single, discrete server action: clearing a player's inventory.

As a subclass of AbstractPlayerCommand, it operates within a framework that automatically handles command registration, permission enforcement, and context provision for actions targeting a specific player entity. Its core responsibility is to translate a high-level command invocation into a direct modification of the game's state.

This class interacts with the world through the server's Entity-Component-System (ECS) architecture. Rather than receiving a direct reference to a Player object, its `execute` method is supplied with ECS handles (Store, Ref, PlayerRef). This decouples the command's logic from the underlying data storage and ensures that all state modifications are performed through the canonical ECS API, maintaining data integrity and consistency within the game loop.

## Lifecycle & Ownership
- **Creation:** A single instance of InventoryClearCommand is instantiated by the server's command registry during the bootstrap phase. The registry scans for command implementations and creates long-lived instances to map command names (e.g., "clear") to executable logic.
- **Scope:** Application-scoped. The single, shared instance persists for the entire lifetime of the server process. The class is designed to be stateless, allowing this single instance to safely service all concurrent or sequential invocations of the command.
- **Destruction:** The instance is destroyed when the command registry is torn down during a graceful server shutdown. No special cleanup is required.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal state, consisting of the command name, description key, and required permission group, is set once in the constructor and is never modified thereafter. The `execute` method does not alter any instance fields.
- **Thread Safety:** Conditionally thread-safe. The class itself contains no mutable state, making the instance inherently safe to share across threads. However, the safety of its execution is dependent on the calling context. The command system **must** ensure that the `execute` method is invoked on the main server thread or within a context that holds the necessary locks on the world's EntityStore. Direct, asynchronous invocation from an arbitrary thread will lead to state corruption and server instability.

## API Surface
The primary contract is the `execute` method, inherited from its parent class. This is the sole entry point for the command's logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Modifies the target player's inventory component by removing all items, where N is the inventory size. Sends a success message via the CommandContext. Throws an assertion error if the target player component cannot be resolved. |

## Integration Patterns

### Standard Usage
Developers do not instantiate or invoke this class directly. The server's command dispatcher handles the entire lifecycle. A player's chat input is parsed, the corresponding command instance is located in the registry, permissions are verified, and the `execute` method is called with the correct world state context.

The following example illustrates the *system's* interaction with the command, not a typical developer use case.

```java
// Conceptual example of the command system invoking the command
CommandParser parser = server.getCommandParser();
CommandInvocation invocation = parser.parse(player, "/clear");

if (invocation.isValid() && permissionManager.hasPermission(player, invocation.getCommand())) {
    // The system provides the necessary ECS context for execution
    invocation.getCommand().execute(invocation.getContext());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new InventoryClearCommand()`. The command system relies on the single, registered instance for mapping and metadata. Creating a new instance bypasses the entire framework.
- **Manual Execution:** Do not call the `execute` method directly. Always dispatch commands through the server's central command service. Bypassing the service will skip critical permission checks and context setup, potentially leading to security vulnerabilities or illegal state modifications.
- **Adding State:** Do not add mutable instance fields to this class or its subclasses. Commands are treated as singletons and must remain stateless to avoid race conditions between different players executing the same command.

## Data Pipeline
The command acts as a state mutator within a larger data flow, translating a player's intent into a change in the ECS database.

> Flow:
> Player Chat Input (`/clear`) -> Network Packet -> Server Command Parser -> Command Registry Lookup -> Permission Check (`GameMode.Creative`) -> **InventoryClearCommand.execute()** -> ECS `Player.getInventory().clear()` -> World State Change -> Inventory Update Packet -> Client-side Inventory Render


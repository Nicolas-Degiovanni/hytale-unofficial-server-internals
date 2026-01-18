---
description: Architectural reference for InventoryItemCommand
---

# InventoryItemCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.inventory
**Type:** Transient

## Definition
```java
// Signature
public class InventoryItemCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The InventoryItemCommand is a server-side, player-specific command responsible for handling interactions with items that function as containers, such as backpacks. It acts as a concrete implementation within the server's Command System framework, inheriting from AbstractPlayerCommand to ensure it is only executed within the context of a valid player entity.

Its primary architectural role is to bridge a player's intent (executing the command) with the server's inventory and user interface systems. When executed, it inspects the player's currently held item in the hotbar. If that item possesses an associated ItemStackItemContainer, the command instructs the player's PageManager to open a new UI window (specifically, Page.Bench) backed by that container. This allows players to interact with nested inventories.

This class directly manipulates high-level server components, including the EntityStore via Refs, the Player component, and its associated Inventory and PageManager, making it a critical link between player input and world state modification.

## Lifecycle & Ownership
- **Creation:** A single instance of InventoryItemCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is not created on a per-request basis.
- **Scope:** The object is a stateless singleton that persists for the entire lifetime of the server. It is registered once and remains available until server shutdown.
- **Destruction:** The instance is garbage collected when the server shuts down and its central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. It contains only static final Message fields, which are immutable constants loaded at class initialization. All operational state (the player, the world, the entity store) is passed into the execute method as arguments, ensuring that each execution is isolated.
- **Thread Safety:** The class itself is thread-safe due to its stateless nature. However, the execute method is designed to be invoked by the server's main thread or a world-specific thread. It operates on player and world components that are not thread-safe and must not be accessed concurrently. The server's command processing queue is responsible for ensuring serialized execution of commands for any given player.

## API Surface
The public API is defined by its parent, AbstractPlayerCommand. The core logic is contained within the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command logic. Validates the player's held item, and if it is a container, sends a request to open its inventory UI. Sends failure messages via the CommandContext if conditions are not met. |

## Integration Patterns

### Standard Usage
A developer or system integrator does not interact with this class directly. The server's command dispatcher invokes it automatically when a player executes the corresponding command (e.g., by typing "/item" in chat). The system is responsible for constructing the CommandContext and providing the necessary entity references.

```java
// Conceptual representation of how the command system invokes this class.
// DO NOT CALL THIS METHOD DIRECTLY.

Command command = commandRegistry.find("item");
if (command instanceof AbstractPlayerCommand) {
    // The system builds the context and references for the player.
    ((AbstractPlayerCommand) command).execute(playerContext, entityStore, playerEntityRef, playerRef, world);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InventoryItemCommand()`. The command must be managed by the server's central command registry to ensure proper lifecycle and visibility.
- **Manual Invocation:** Calling the `execute` method manually bypasses critical infrastructure such as permission checks, argument parsing, and context validation. Always let the server's command dispatcher handle invocation.

## Data Pipeline
The command is triggered by player input and results in a potential UI update packet being sent back to the client. The data flow is unidirectional from the client's request to the client's UI update.

> Flow:
> Client Chat Input -> Network Packet -> Server Command Parser -> **InventoryItemCommand.execute()** -> PageManager -> Network Packet (Open Window) -> Client UI Update


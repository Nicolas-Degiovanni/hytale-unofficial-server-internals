---
description: Architectural reference for InventoryCommand
---

# InventoryCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.inventory
**Type:** Transient

## Definition
```java
// Signature
public class InventoryCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The InventoryCommand class is a **Command Group** that acts as a structural node within the server's command processing system. It does not implement any direct game logic. Its sole responsibility is to aggregate and provide a unified entry point for all inventory-related sub-commands, such as clearing an inventory or inspecting another player's items.

Architecturally, this class embodies the **Composite Design Pattern**. It is a non-terminal expression in the command grammar, composing other command objects (InventoryClearCommand, InventorySeeCommand, etc.) into a hierarchical tree. When the server's CommandSystem parses a command like `/inventory clear`, it first resolves the `inventory` token to this InventoryCommand instance. This object then delegates the remainder of the parsing and execution to the appropriate registered sub-command, in this case, InventoryClearCommand.

This pattern decouples the parent command from its children, allowing new inventory-related sub-commands to be added with no modification to the InventoryCommand class itself, promoting extensibility and maintainability.

## Lifecycle & Ownership
- **Creation:** An instance of InventoryCommand is created by the server's central CommandRegistry during the server bootstrap phase. This process typically involves reflection-based scanning of designated packages for command classes.
- **Scope:** Application-scoped. Once instantiated and registered, the object persists for the entire lifetime of the server session. It is a stateless router whose configuration is fixed at startup.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** The state of an InventoryCommand instance is configured entirely within its constructor and is considered **effectively immutable** post-initialization. The state consists of its primary name, aliases, and a collection of its child sub-commands. There are no mechanisms for modifying this state at runtime.
- **Thread Safety:** This class is inherently thread-safe. Its immutable-after-construction nature ensures that multiple threads can safely access it for command lookups without locks or synchronization. The responsibility for thread-safe *execution* of the final command logic lies with the sub-command itself and the server's command execution engine.

## API Surface
The public contract of this class is minimal and is entirely focused on its construction-time configuration. There are no public methods intended for invocation after the object has been registered with the CommandSystem.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| InventoryCommand() | Constructor | O(1) | Initializes the command group, setting its primary name ("inventory"), alias ("inv"), and registering all constituent sub-commands. |

## Integration Patterns

### Standard Usage
A developer does not interact with an instance of this class directly. The class is designed to be discovered and registered automatically by the server's command handling framework. To add a new sub-command, a developer would create a new command class and add it to the constructor's chain of `addSubCommand` calls.

```java
// Example of adding a new sub-command during development
public InventoryCommand() {
   super("inventory", "server.commands.inventory.desc");
   this.addAliases("inv");
   this.addSubCommand(new InventoryClearCommand());
   this.addSubCommand(new InventorySeeCommand());
   this.addSubCommand(new InventoryItemCommand());
   this.addSubCommand(new InventoryBackpackCommand());
   // A new command is added here
   this.addSubCommand(new InventorySortCommand());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new InventoryCommand()` in game logic. Doing so creates an orphan object that is not registered with the server's CommandSystem and will have no effect.
- **Runtime Modification:** Do not attempt to get an instance of this class from the CommandRegistry to call `addSubCommand` at runtime. The command tree is intended to be static after server startup, and modifying it can lead to race conditions and undefined behavior.

## Data Pipeline
InventoryCommand serves as a routing step in the command processing pipeline. It validates the initial part of the command string and directs the flow to the correct handler.

> Flow:
> Player Chat Input (`/inventory clear <player>`) -> Network Packet -> Server Command Parser -> **InventoryCommand** (Resolves `inventory` token) -> InventoryClearCommand (Resolves `clear` token and executes logic) -> Player Manager -> Player State Update -> Network Response


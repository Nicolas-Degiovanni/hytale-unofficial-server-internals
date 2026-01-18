---
description: Architectural reference for TeleportHistoryCommand
---

# TeleportHistoryCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Transient

## Definition
```java
// Signature
public class TeleportHistoryCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The TeleportHistoryCommand is a concrete implementation of the Command pattern, designed to operate within the server's command dispatch system. It serves as a leaf node in the command processing tree, responsible for handling the user-facing `/teleport history` command.

Architecturally, this class acts as a thin bridge between the user interface layer (player chat commands) and the underlying Entity Component System (ECS). Its sole responsibility is to query the **TeleportHistory** component associated with the executing player's entity, format the data into a user-readable message, and send it back to the player via the CommandContext. It is a read-only command that inspects game state without modifying it.

This command is an integral part of the **TeleportPlugin**, demonstrating how plugin-specific logic is exposed to players in a structured and permission-controlled manner.

### Lifecycle & Ownership
- **Creation:** A single instance of TeleportHistoryCommand is instantiated by its parent, the **TeleportPlugin**, during the server's plugin initialization phase. It is then registered with the central command system.
- **Scope:** The object instance persists for the entire lifecycle of the TeleportPlugin. It is effectively a singleton within the scope of the command registry. However, its execution context, provided via the *execute* method parameters, is transient and unique to each command invocation.
- **Destruction:** The instance is destroyed and becomes eligible for garbage collection when the TeleportPlugin is disabled or the server shuts down, at which point it is deregistered from the command system.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. Its fields are static final Message templates, which are immutable. All state required for execution, such as the target player and the entity store, is passed as arguments to the *execute* method. This design ensures that a single instance can safely handle concurrent command executions if the framework were to ever support it.

- **Thread Safety:** This class is inherently thread-safe due to its stateless nature. The responsibility for ensuring safe access to the ECS data (Store, Ref) lies with the calling command execution framework. The framework guarantees that the *execute* method is invoked in a context where operations on the provided EntityStore are safe.

## API Surface
The public API is defined by its abstract parent and is not intended for direct invocation by developers. The framework calls the *execute* method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Framework entry point. Retrieves teleport history counts from the player's entity and sends a formatted message. This operation is constant time, relying on direct component lookups. |

## Integration Patterns

### Standard Usage
A developer does not call this class directly. Instead, an instance is registered with the command system, typically within a plugin's lifecycle. The system then invokes it based on player input.

```java
// Example of registering a similar command within a plugin
// This is a conceptual example. The actual API may differ.

public class MyPlugin extends JavaPlugin {
    @Override
    public void onEnable() {
        // The command system handles the lifecycle from here.
        getCommandRegistry().register(new TeleportHistoryCommand());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Execution:** Never create an instance of this command and call *execute* manually. Doing so bypasses critical framework services, including permission validation, argument parsing, and context setup. This will lead to NullPointerExceptions and an unstable server state.

- **Storing State:** Do not add mutable instance fields to this class. The command registry may reuse the instance. Storing per-execution state on the instance can create race conditions and bleed state between different players' command invocations.

## Data Pipeline
The flow of data for this command is unidirectional and initiated by the player.

> Flow:
> Player Input (`/teleport history`) → Server Command Dispatcher → **TeleportHistoryCommand.execute()** → EntityStore Component Lookup → Read TeleportHistory State → Message Formatting → CommandContext.sendMessage() → Network Packet → Client Chat UI

---


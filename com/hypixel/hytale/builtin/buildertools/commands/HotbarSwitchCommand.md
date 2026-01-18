---
description: Architectural reference for HotbarSwitchCommand
---

# HotbarSwitchCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class HotbarSwitchCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The HotbarSwitchCommand class is a server-side implementation of the Command Pattern, designed to handle player-initiated hotbar management. It serves as the entry point for the `/hotbar` chat command, providing a bridge between the server's command processing system and the player's underlying Entity-Component-System (ECS) data.

This command is a thin controller. Its sole responsibility is to parse and validate user input—specifically a target slot and an optional save flag—and delegate the core business logic to the HotbarManager component. It does not contain any state manipulation logic itself. By extending AbstractPlayerCommand, it automatically benefits from foundational infrastructure that ensures a command is executed by a valid, authenticated player entity.

Architecturally, it is a leaf node in the server's input processing tree, activated only when its registered name, "hotbar", is matched by the CommandSystem. Its existence is tightly coupled to the Creative game mode, enforcing a clear separation of concerns between survival and building mechanics.

## Lifecycle & Ownership
- **Creation:** A single instance of HotbarSwitchCommand is instantiated by the server's command registration system during the server bootstrap sequence or when the parent module (BuilderTools) is loaded. It is not created on a per-request basis.
- **Scope:** The command object is a long-lived singleton for the duration of the server's runtime. It is stored in a central command registry and is reused for every execution of the `/hotbar` command.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down or the command is explicitly unregistered from the CommandSystem.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The fields `hotbarSlotArg` and `saveInsteadOfLoadArg` are not execution-specific state; they are immutable definitions that describe the command's argument structure to the parsing system. All state required for execution (the player, the world, the arguments) is passed into the `execute` method via the CommandContext.
- **Thread Safety:** This class is thread-safe. The command's `execute` method is invoked synchronously on the main server thread (the "tick" thread). As the object holds no mutable state, there are no risks of race conditions within this class. Concurrency guarantees for the underlying ECS operations are the responsibility of the `Store` and `HotbarManager`.

## API Surface
The primary contract is the `execute` method, which is an override. The constructor is intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command logic. Delegates to HotbarManager to save or load a hotbar. Complexity is constant time relative to this class, but depends on the underlying ECS implementation. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by developers. It is automatically invoked by the server's command system in response to player input. The standard interaction is a player typing the command in chat.

*Example player interaction:*
> /hotbar 3
> *Loads hotbar from slot 3.*

> /hotbar 5 --save
> *Saves current hotbar to slot 5.*

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new HotbarSwitchCommand()`. The server's command registry manages the lifecycle of command objects. Direct instantiation will result in a non-functional object that is not registered to handle any chat commands.
- **Manual Execution:** Do not call the `execute` method directly. This bypasses critical infrastructure, including permission checks, argument validation, and context setup. Manually constructing the required `CommandContext` and ECS `Store` and `Ref` objects is complex and error-prone.

## Data Pipeline
This component acts as a control endpoint, not a data processing stage. The flow is initiated by user action and results in a state change within the ECS.

> Flow:
> Player Chat Input -> Server Network Listener -> CommandSystem Parser -> **HotbarSwitchCommand.execute()** -> Player.getHotbarManager() -> HotbarManager.saveHotbar() / loadHotbar() -> EntityStore State Mutation -> Network Packet (to sync client hotbar)


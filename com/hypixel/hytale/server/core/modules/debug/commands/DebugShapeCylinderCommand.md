---
description: Architectural reference for DebugShapeCylinderCommand
---

# DebugShapeCylinderCommand

**Package:** com.hypixel.hytale.server.core.modules.debug.commands
**Type:** Transient Command Object

## Definition
```java
// Signature
public class DebugShapeCylinderCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DebugShapeCylinderCommand is a concrete implementation of the **Command Pattern**, designed to integrate with the server's command processing system. It serves as a specific, user-invokable action within the broader debug module.

Its primary architectural role is to act as a translator between a high-level player input (a chat command) and a low-level engine function (rendering a debug shape). It queries the server's Entity-Component-System (ECS) via the EntityStore to retrieve the player's current world position from their TransformComponent. It then delegates the actual rendering logic to the DebugUtils service, effectively decoupling the command parsing layer from the debug visualization system.

This class is not intended for direct invocation by game logic; it is exclusively discovered and managed by the server's central command registry.

### Lifecycle & Ownership
- **Creation:** A single instance of this command is instantiated by the server's command registration system during the server bootstrap phase. It is discovered alongside other command classes and registered with a central command manager.
- **Scope:** The command object is a long-lived singleton for the duration of the server's runtime. It is held in memory by the command registry.
- **Destruction:** The instance is destroyed when the server shuts down and the command registry is cleared. The execution context itself is ephemeral, lasting only for the duration of the `execute` method call.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields. All necessary state, such as the player reference, world context, and entity store, is provided as arguments to the `execute` method. The `MESSAGE_COMMANDS_DEBUG_SHAPE_CYLINDER_SUCCESS` field is a static, immutable constant.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the `execute` method is designed to be invoked exclusively from the main server thread or a dedicated world-update thread. The objects passed as parameters, particularly `World` and `Store`, are not guaranteed to be thread-safe. Direct invocation from asynchronous tasks will lead to race conditions and world corruption. The use of `ThreadLocalRandom` further implies a single-threaded execution model.

## API Surface
The public API is minimal, designed for framework integration rather than direct developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DebugShapeCylinderCommand() | constructor | O(1) | Registers the command's name and description key with the parent class. Intended for framework use only. |
| execute(...) | protected void | O(1) | The command's entry point. Retrieves the player's position and delegates to DebugUtils to render a cylinder. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class programmatically. This command is invoked by a player through the in-game chat or console. The framework handles the entire lifecycle of parsing the input and dispatching it to this class.

The intended interaction is a player typing the following command in-game:
```
/debug shape cylinder
```
This action triggers the `execute` method on the server, resulting in a randomly colored debug cylinder appearing at the player's location.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DebugShapeCylinderCommand()`. The command system handles instantiation and registration. Manual creation will result in a non-functional command that is not registered to any chat input.
- **Manual Invocation:** Never call the `execute` method directly. This bypasses critical framework infrastructure, including permission checks, context validation, and argument parsing. Bypassing the framework can lead to NullPointerExceptions and unstable server state.

## Data Pipeline
The flow of data for this command begins with a player action and terminates with a world state change and a network response.

> Flow:
> Player Chat Packet -> Network Layer -> Command Dispatcher -> **DebugShapeCylinderCommand.execute()** -> EntityStore Query -> DebugUtils.addCylinder() -> World Render Queue -> CommandContext.sendMessage() -> Network Response Packet


---
description: Architectural reference for TeleportTopCommand
---

# TeleportTopCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Transient

## Definition
```java
// Signature
public class TeleportTopCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The TeleportTopCommand class implements the server-side logic for the `/top` command. It is a concrete implementation within the server's command processing framework, inheriting from AbstractPlayerCommand to ensure it is executed within the context of a specific player entity.

Architecturally, this class serves as a command handler that translates a player's intent into a state change within the Entity Component System (ECS). It does not directly manipulate the player's position. Instead, it follows a deferred execution pattern by creating and attaching a **Teleport** component to the player entity. A separate, dedicated system (e.g., a TeleportSystem) is responsible for processing this component on a subsequent server tick, thereby decoupling the command logic from the core physics and entity update loops.

This command interacts with several key engine systems:
-   **World:** To query chunk data and determine the highest solid block at the player's current X/Z coordinates.
-   **EntityStore (ECS):** To read the player's current TransformComponent and HeadRotation, and to write the new Teleport and TeleportHistory components.
-   **CommandContext:** To send feedback messages back to the executing player.

### Lifecycle & Ownership
-   **Creation:** A single instance of TeleportTopCommand is created by the server's command registration system during the server bootstrap sequence. It is discovered and registered to handle the "top" command identifier.
-   **Scope:** The command object is a long-lived singleton that persists for the entire server session. However, its execution context is transient and scoped to a single invocation of the `execute` method.
-   **Destruction:** The instance is garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is stateless. It contains no mutable instance fields. All state required for execution (the player, the world, the entity store) is passed as arguments to the `execute` method. The static Message fields are immutable constants initialized at class-loading time.
-   **Thread Safety:** The object itself is inherently thread-safe due to its stateless nature. The Hytale server architecture guarantees that the `execute` method is invoked on the main server thread for the corresponding world. Therefore, operations on the World and EntityStore (via the `store` parameter) are safe within the scope of this method.

**WARNING:** Manually invoking the `execute` method from an asynchronous thread without external synchronization on the World and EntityStore will lead to severe data corruption and server instability.

## API Surface
The public contract is defined by its inheritance from AbstractPlayerCommand. The primary entry point is the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command logic. Reads the player's position, finds the highest block in the column, records the previous location, and attaches a Teleport component to the player entity to initiate the move. Fails if the target chunk is not loaded in memory. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the server's command system when a player issues the `/top` command. The system provides the required context for execution.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TeleportTopCommand()`. The command will not be registered with the server and will have no effect.
-   **Manual Invocation:** Do not call the `execute` method directly. The `CommandContext`, `Store`, and `Ref` parameters are tightly coupled to the engine's internal state for a specific tick and cannot be safely or easily constructed. Bypassing the command system breaks permissions checks and can lead to an inconsistent world state.

## Data Pipeline
The execution of this command initiates a clear, component-driven data flow that defers the final state change to a dedicated system.

> Flow:
> Player Chat Input (`/top`) -> Command Parser -> **TeleportTopCommand.execute()** -> Add `Teleport` Component to Player Entity -> Teleport System (processes component on next tick) -> Modifies Player's `TransformComponent` -> Player position is updated.


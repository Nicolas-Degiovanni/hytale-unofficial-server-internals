---
description: Architectural reference for RotateCommand
---

# RotateCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Configuration Object

## Definition
```java
// Signature
public class RotateCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The RotateCommand class is a command definition that exposes the `/rotate` server command for players with creative mode permissions. It serves as the primary interface between player chat input and the backend Builder Tools system.

Architecturally, this class follows the **Composite Command Pattern**. It is not a single command but a collection (`AbstractCommandCollection`) that houses multiple "variants", each representing a different way to use the `/rotate` command (e.g., by axis and angle, or by arbitrary yaw, pitch, and roll).

Its core responsibility is to define the command structure, parse player-provided arguments, and perform initial validation. Crucially, it **does not contain the block rotation logic itself**. Instead, it acts as a dispatcher, validating the request and then delegating the operation to the `BuilderToolsPlugin` work queue. This decouples the command parsing and user interface layer from the core world-manipulation engine, ensuring that complex operations are handled by the appropriate system during the main game tick.

## Lifecycle & Ownership

-   **Creation:** A single instance of RotateCommand is created during the server's bootstrap sequence, typically as part of the `BuilderToolsPlugin` initialization. It is then registered with the server's central command system.
-   **Scope:** The command definition object is long-lived, persisting for the entire server session. It holds no per-player state and is safe to be used as a shared, immutable definition.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the `BuilderToolsPlugin` is unloaded or the server shuts down.

## Internal State & Concurrency

-   **State:** This class and its inner variants are effectively **stateless**. They are configuration objects whose internal fields define the command's arguments and structure. This state is immutable after construction. All runtime data, such as the player executing the command or the world state, is passed into the `execute` method via its parameters.
-   **Thread Safety:** Command execution is managed by the server's command processing system, which typically operates on the main server thread. The class is inherently thread-safe due to its stateless design. The delegation to `BuilderToolsPlugin.addToQueue` ensures that the potentially long-running rotation task is handed off to a system designed to manage such operations without blocking the command processor.

## API Surface

The primary API is the command's execution contract, fulfilled by the private inner classes. These methods are invoked by the command system, not by developers directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RotateArbitraryVariant.execute(...) | void | O(1) | Parses yaw, pitch, and roll. Validates player selection and queues an arbitrary rotation task with the BuilderToolsPlugin. |
| RotateAxisVariant.execute(...) | void | O(1) | Parses axis and a 90-degree angle. Validates player selection and queues an axis-aligned rotation task with the BuilderToolsPlugin. |

**Warning:** The O(1) complexity only reflects the command's immediate action, which is to queue a task. The actual rotation operation performed by the `BuilderToolsPlugin` is O(N), where N is the number of blocks in the player's selection.

## Integration Patterns

### Standard Usage

Developers do not interact with this class directly. Its functionality is exposed to players through the in-game chat. The key integration pattern is its delegation to the `BuilderToolsPlugin`, which is the correct way to enqueue a builder operation.

The core logic within the `execute` method demonstrates this pattern:

```java
// Conceptual example of the delegation pattern
// This code runs when a player executes the command.

// 1. Validate the player's context
if (PrototypePlayerBuilderToolSettings.isOkayToDoCommandsOnSelection(ref, playerComponent, store)) {
    
    // 2. Parse arguments from the command context
    int angle = this.angleArg.get(context);
    Axis axis = this.axisArg.get(context);

    // 3. Create a task (lambda) representing the work
    // WARNING: The task does not execute immediately.
    var rotationTask = (r, s, componentAccessor) -> s.rotate(r, axis, angle, componentAccessor);

    // 4. Delegate the task to the central plugin queue
    BuilderToolsPlugin.addToQueue(playerComponent, playerRef, rotationTask);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new RotateCommand()` in your own code. Command objects must be managed and registered by the server's command system, typically via a central plugin entry point.
-   **Direct Execution:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, argument parsing, and context provision, which will lead to runtime exceptions and unpredictable behavior.
-   **Implementing Rotation Logic:** Do not replicate the logic of this command to perform rotations. Always delegate world modification tasks to the appropriate manager, like the `BuilderToolsPlugin`, to ensure operations are queued and executed safely within the main game loop.

## Data Pipeline

The flow of data for a rotate command begins with player input and ends with a world update, with RotateCommand acting as an early validation and dispatching stage.

> Flow:
> Player Chat Input (`/rotate 90`) -> Server Command Parser -> **RotateCommand** (Variant Selection & Argument Parsing) -> `BuilderToolsPlugin.addToQueue` -> Game Tick Work Queue -> `Selection.rotate()` -> World State Modification -> Network Sync to Clients


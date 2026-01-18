---
description: Architectural reference for FlipCommand
---

# FlipCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient Handler

## Definition
```java
// Signature
public class FlipCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The FlipCommand class is an implementation of the Command design pattern, serving as a direct interface between the player's chat input and the server's Builder Tools system. It is responsible for parsing and validating the `/flip` command, which allows players in Creative mode to mirror a selected region of blocks along a specific axis.

Architecturally, this command acts as a translator and a delegator. It does not contain the logic for the world manipulation itself. Instead, its primary responsibilities are:
1.  **Command Registration:** It registers the `flip` command keyword and its argument variants with the server's command system.
2.  **Context Unpacking:** Upon invocation, it unpacks the `CommandContext` to retrieve references to the player, the world, and the Entity Component System (ECS) store.
3.  **State Retrieval:** It queries the ECS for necessary components, such as the player's `HeadRotation`, to determine the axis of the flip operation.
4.  **Deferred Execution:** It delegates the core flip logic to the `BuilderToolsPlugin` by adding the operation to a processing queue. This decouples the immediate command invocation from the potentially resource-intensive world modification, preventing server stalls.

The class utilizes a nested private class, `FlipWithDirectionCommand`, to handle command variants with explicit direction arguments (e.g., `/flip up`), providing a clean structure for command overloading.

### Lifecycle & Ownership
-   **Creation:** A single instance is created by the `BuilderToolsPlugin` during its initialization phase. This instance is then registered with the server's central command registry.
-   **Scope:** The object's lifecycle is tied to the `BuilderToolsPlugin`. It persists as long as the plugin is active and the command is registered.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the `BuilderToolsPlugin` is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is stateless. It does not hold any mutable data between invocations. All required information is passed into the `execute` method via the `CommandContext` and ECS `Store`.
-   **Thread Safety:** Conditionally Safe. The `execute` method is designed to be called by the server's main command processing thread. While the class itself is stateless, the underlying operation it queues is not. The `BuilderToolsPlugin` queue is responsible for ensuring that world modifications are executed in a thread-safe, serialized manner. Direct, concurrent invocation of the `execute` method from multiple threads is not supported and will lead to undefined behavior.

## API Surface
The public API is minimal, intended for use only by the server's command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FlipCommand() | constructor | O(1) | Initializes the command, registers its name, and sets up its permission requirements. |
| execute(...) | protected void | O(1) | Entry point for command execution. Validates context and queues the flip operation. Complexity is for queuing only; the actual world operation is O(N). |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly via code. It is invoked by the game engine in response to a player's chat input. The system handles parsing, permission checks, and invocation.

A player in-game would use it as follows:
-   `/flip` - Flips the current selection based on the direction the player is looking.
-   `/flip north` - Flips the current selection along the north-south axis.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new FlipCommand()` in your own code. Command objects must be managed and registered by the server's command system to function correctly.
-   **Manual Invocation:** Do not call the `execute` method directly. Bypassing the command system will result in a missing or incomplete `CommandContext`, leading to NullPointerExceptions and bypassing critical permission checks.

## Data Pipeline
The flow of data for a flip operation is initiated by the player and travels through several server systems before resulting in a world change.

> Flow:
> Player Chat Input -> Network Layer -> Server Command Parser -> **FlipCommand.execute** -> ECS Component Query (for HeadRotation) -> BuilderToolsPlugin Queue -> World Edit Operation -> World State Update -> Network Packet (to clients)


---
description: Architectural reference for ShiftCommand
---

# ShiftCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Singleton

## Definition
```java
// Signature
public class ShiftCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ShiftCommand class is a server-side, player-executable command that implements the Command Pattern. It is registered with the server's central command system and serves as the entry point for the `/shift` builder tool functionality.

Its primary architectural role is to act as a translator between raw player chat input and a structured, asynchronous world modification task. It is responsible for parsing arguments (distance and axis), validating the player's state and permissions, and determining the direction of the shift operation based on either an explicit axis or the player's head rotation.

Crucially, this command does not directly manipulate the world state. Instead, it delegates the complex and potentially long-running operation to the BuilderToolsPlugin work queue. This decouples the immediate command parsing logic from the core world-editing subsystem, preventing the command execution from blocking the main server thread.

## Lifecycle & Ownership
- **Creation:** A single instance of ShiftCommand is created by the server's command registry during the server bootstrap phase. It is discovered and instantiated alongside all other built-in commands.
- **Scope:** The singleton instance persists for the entire lifetime of the server process. It is a stateless service object whose `execute` method is called in response to player actions.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The member fields `distanceArg` and `axisArg` are not state variables; they are immutable definitions for the command argument parser, configured once in the constructor. All state required for an execution, such as the player reference and command arguments, is passed into the `execute` method.

- **Thread Safety:** The ShiftCommand instance itself is thread-safe due to its stateless nature. However, the `execute` method is designed to be called from a specific server thread that has access to the game state. The operations it initiates are fundamentally unsafe to perform concurrently.

    **WARNING:** The class achieves safety by dispatching the world modification logic to the `BuilderToolsPlugin` queue. This ensures the actual `shift` operation is executed serially and on the correct thread, preventing race conditions and world corruption. Direct modification of the world from within the `execute` method would violate engine concurrency contracts.

## API Surface
The public API is minimal, as interaction is intended to occur through the server's command system, not direct method invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses arguments and enqueues a world modification task. Throws exceptions on invalid arguments. The complexity of the enqueued task is variable and depends on selection size. |

## Integration Patterns

### Standard Usage
This class is not designed for direct programmatic use. A developer or user interacts with it by typing the command into the game client. The server's command system handles parsing and invocation.

*Example player interaction:*
```plaintext
// Shifts the player's selection 10 blocks along the positive Y axis (up).
/shift 10 UP

// Shifts the player's selection 5 blocks in the direction the player is looking.
/shift 5
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ShiftCommand()`. The server's command system manages the lifecycle of this singleton object. Manually creating an instance will result in an un-registered and non-functional command.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the server's permission system, argument parsing pipeline, and network synchronization, which can lead to inconsistent state or security vulnerabilities.
- **Assuming Synchronous Execution:** The `execute` method returns immediately after queueing the task. Any code following a command dispatch should not assume the world modification has already completed.

## Data Pipeline
The flow of data for a shift operation begins with the player and ends with a world state change, with ShiftCommand acting as the initial processing and dispatching agent.

> Flow:
> Player Input (`/shift 10`) -> Network Packet -> Server Command Parser -> **ShiftCommand.execute()** -> Validation & Direction Calculation -> BuilderToolsPlugin Work Queue -> World Update Thread -> `EntityStore.shift()` -> World State Modified

---


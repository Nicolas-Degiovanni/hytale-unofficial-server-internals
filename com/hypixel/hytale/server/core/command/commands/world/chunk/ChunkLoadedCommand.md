---
description: Architectural reference for ChunkLoadedCommand
---

# ChunkLoadedCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkLoadedCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ChunkLoadedCommand is a concrete implementation of the Command Pattern, designed to operate within the server's administrative command processing system. Its primary function is to serve as a diagnostic tool, allowing privileged users to query the state of a player's currently loaded world chunks.

Architecturally, this class acts as a bridge between the command input system and the server's core Entity-Component-System (ECS). It does not contain any logic for chunk management itself. Instead, it queries the `ChunkTracker` component, which is the authoritative source for a player's chunk data.

By extending `AbstractTargetPlayerCommand`, it inherits the complex logic for parsing and resolving a target player from command arguments. This specialization allows ChunkLoadedCommand to focus exclusively on its core task: retrieving and reporting data, adhering to the Single Responsibility Principle.

## Lifecycle & Ownership
- **Creation:** A single instance of ChunkLoadedCommand is instantiated by the server's command registry during the bootstrap phase. The system scans for command classes and registers them for the server's operational lifetime.
- **Scope:** The command object itself is a long-lived singleton, persisting for the entire server session. It exists to serve requests but holds no per-request state.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It possesses no internal fields to store data between invocations. Its behavior is determined entirely by the arguments supplied to the `execute` method at runtime. This design is critical for command objects to ensure reusability and prevent side effects.
- **Thread Safety:** The class is inherently **thread-safe** due to its stateless nature. Multiple threads can invoke the `execute` method on the shared instance without risk of data corruption. However, the command is expected to be executed on the main server thread, as the ECS components it interacts with, such as `ChunkTracker` and `EntityStore`, are typically not designed for concurrent access.

## API Surface
The primary public contract is its registration with the command system via its constructor. The `execute` method is the operational entry point, but it is a protected method intended for framework invocation only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | **Framework Override.** Retrieves the ChunkTracker from the target player entity and uses the CommandContext to send a formatted message back to the command issuer. Throws assertion errors if the target entity lacks a ChunkTracker. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked implicitly when a user with appropriate permissions executes the corresponding command in-game or via the server console. The framework handles parsing, dispatch, and context provision.

A conceptual view of the framework's interaction:

```java
// PSEUDOCODE: How the command system might invoke this class
CommandSystem commandSystem = server.getCommandSystem();
CommandContext context = create_context_from_user_input("/chunk loaded SomePlayer");

// The system finds the registered command and executes it
// Do NOT replicate this logic.
commandSystem.dispatch(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkLoadedCommand()`. An instance created this way will not be registered with the server's command system and will be non-functional.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the entire command processing pipeline, including critical steps like permission checking, argument validation, and context setup. This can lead to unpredictable behavior and NullPointerExceptions.

## Data Pipeline
The flow of data for a typical invocation is a simple request-response cycle orchestrated by the server's command framework.

> Flow:
> Player Command Input -> Network Packet -> Command Parser -> **ChunkLoadedCommand** -> EntityStore Query -> ChunkTracker Component -> Formatted String -> CommandContext -> Network Packet -> Player Chat UI


---
description: Architectural reference for ClearEditHistory
---

# ClearEditHistory

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class ClearEditHistory extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ClearEditHistory class is a concrete implementation of the Command Pattern, designed to integrate with the server's core command processing system. It serves as a dedicated handler for the `/clearEditHistory` console command and its registered aliases.

This class acts as a thin translation layer, converting a player-initiated command into a specific action within the BuilderToolsPlugin. By extending AbstractPlayerCommand, it inherits the necessary infrastructure to ensure the command is executed by a valid player entity and not, for example, the server console. The framework also enforces the configured permission group, restricting this command's execution to players in Creative mode.

Its primary architectural role is to decouple the command input system from the business logic of the builder tools. It receives a fully hydrated execution context and delegates the core operation to the appropriate subsystem, in this case, the BuilderToolsPlugin's asynchronous task queue.

### Lifecycle & Ownership
-   **Creation:** An instance of ClearEditHistory is created by the server's command registration system during the plugin loading phase. The system discovers command classes and instantiates them to build a central command registry.
-   **Scope:** The singleton instance is held by the command registry and persists for the entire lifecycle of the server or until the parent BuilderToolsPlugin is unloaded.
-   **Destruction:** The object is marked for garbage collection when the server shuts down or the plugin is disabled, at which point the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is entirely dependent on the parameters passed to its `execute` method.
-   **Thread Safety:** The `execute` method is designed to be called exclusively from the main server thread as part of the game loop's command processing tick. While the method itself is inherently thread-safe due to its statelessness, the components it interacts with (Store, Ref, PlayerRef) are not.

    **Warning:** The core operation is intentionally offloaded to the BuilderToolsPlugin queue via `addToQueue`. This is a critical concurrency pattern to prevent long-running builder tool operations from blocking the main server thread. Direct manipulation of the EntityStore from this class would violate this design.

## API Surface
The public API is minimal and intended for framework use only. Direct invocation by developer code is a severe anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Framework-invoked method. Retrieves the player component and enqueues a `clearHistory` task in the BuilderToolsPlugin. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. It is invoked automatically by the server's command system when a player with appropriate permissions executes one of the associated commands in their client console.

*Example Player Input:*
```
/clearEditHistory
```
or
```
/clearHistory
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ClearEditHistory()`. The command framework is solely responsible for the lifecycle of command objects.
-   **Manual Invocation:** Do not call the `execute` method directly. Doing so bypasses the entire command processing pipeline, including critical permission checks, argument parsing, and context validation. This can lead to server instability or security vulnerabilities.

## Data Pipeline
The flow of data for this command begins with player input and terminates with a modification to the world's entity storage. The ClearEditHistory class is a single, critical step in this pipeline responsible for delegation.

> Flow:
> Player Command Input -> Server Command Parser -> Permission Check -> **ClearEditHistory.execute()** -> BuilderToolsPlugin Task Queue -> EntityStore.clearHistory() -> World State Update


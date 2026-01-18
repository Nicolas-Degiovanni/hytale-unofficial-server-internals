---
description: Architectural reference for WorldPerfResetCommand
---

# WorldPerfResetCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world.perf
**Type:** Service Component

## Definition
```java
// Signature
public class WorldPerfResetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldPerfResetCommand class is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. It serves as a specific, user-invokable action within the broader command hierarchy, residing logically at the path `/world perf reset`.

Its primary architectural role is to provide a controlled entry point for administrative users to reset performance monitoring data. It decouples the user input (a typed command) from the underlying system action (invoking `clearMetrics` on one or more World instances).

The command exhibits two scopes of operation, controlled by the presence of a command-line flag:
1.  **Targeted Scope:** The default behavior operates on the World instance associated with the command's execution context.
2.  **Global Scope:** When the `--all` flag is provided, the command escalates its operation, querying the global Universe service to affect *every* active World instance. This demonstrates a common pattern of escalating impact based on explicit user intent.

## Lifecycle & Ownership
-   **Creation:** A single instance of WorldPerfResetCommand is instantiated by the server's `CommandRegistry` during the server bootstrap sequence. The system scans for classes extending command base types and registers them for the server's lifetime.
-   **Scope:** The object is a singleton within the scope of the `CommandRegistry`. It persists for the entire server session. It is **not** created on a per-request basis.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the `CommandRegistry` is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its instance fields, such as `allFlag`, are configured once in the constructor and are not mutated during command execution. The state that it modifies is entirely external, belonging to the `World` and `Universe` objects.

-   **Thread Safety:** The `execute` method is designed to be called by the server's main command processing thread. It is not designed for concurrent invocation. **WARNING:** While the command class itself is stateless, the operations it performs on the `Universe` and `World` objects are not inherently thread-safe. It relies on the assumption that the underlying `Universe.getWorlds()` and `World.clearMetrics()` methods are synchronized or are otherwise safe to call from the server's main thread. Developers must ensure the thread safety of the underlying world systems, not this command class.

## API Surface
The public contract is almost exclusively defined by its role as a command handler. Direct invocation is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N) | Executes the performance metric reset logic. Complexity is O(1) for a single world, or O(N) when the `--all` flag is used, where N is the number of loaded worlds. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked automatically by the server's command handler when a user with appropriate permissions executes the corresponding command.

The conceptual flow is initiated by user input:

```sh
# Console or in-game chat
/world perf reset
/world perf reset --all
```

This input is then routed by the command system to the `execute` method of the registered WorldPerfResetCommand instance.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldPerfResetCommand()`. The command system is responsible for the lifecycle of command objects. Direct instantiation creates an unmanaged object that is not registered to handle any user input.

-   **Manual Invocation:** Avoid calling the `execute` method directly. This bypasses the command system's critical infrastructure, including permission checks, context validation, and argument parsing. Bypassing the system can lead to NullPointerExceptions or inconsistent state.

## Data Pipeline
This component does not process a data stream. Instead, it represents a control flow initiated by a user action.

> Flow:
> User Input (`/world perf reset`) -> Command Parser -> **WorldPerfResetCommand.execute()** -> World.clearMetrics() -> User Feedback Message

> Flow (Global):
> User Input (`/world perf reset --all`) -> Command Parser -> **WorldPerfResetCommand.execute()** -> Universe.getWorlds() -> (For each World) -> World.clearMetrics() -> User Feedback Message


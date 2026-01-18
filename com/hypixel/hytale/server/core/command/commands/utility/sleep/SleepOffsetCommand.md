---
description: Architectural reference for SleepOffsetCommand
---

# SleepOffsetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.sleep
**Type:** Transient

## Definition
```java
// Signature
public class SleepOffsetCommand extends CommandBase {
```

## Architecture & Concepts
The SleepOffsetCommand is a concrete implementation of the Command Pattern, designed to provide administrators with runtime control over the server's core ticking mechanism. Its primary architectural function is to serve as a safe and managed bridge between the user-facing command system and the low-level, performance-critical TickingThread.

This command directly manipulates the static SLEEP_OFFSET field within TickingThread. This field is a tuning parameter that adjusts the sleep duration within the main server loop, effectively controlling CPU usage and tick rate precision. By encapsulating this modification within a command, the system avoids exposing the TickingThread directly and allows for permission checks, argument parsing, and user feedback.

This class is not a service or a manager; it is a stateless executor that translates a user's text-based intention into a direct mutation of a global engine variable.

## Lifecycle & Ownership
-   **Creation:** An instance of SleepOffsetCommand is created by the server's central command registration service during the bootstrap sequence. The system scans for classes extending CommandBase and instantiates them to build the command registry.
-   **Scope:** The object instance persists for the entire server session, held as a reference within the command registry. However, it holds no state between individual executions.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The SleepOffsetCommand instance is stateless. Its fields, percentFlag and offsetArg, are final definitions that describe the command's argument structure, not mutable data. The state it modifies is **external and global**: the static field TickingThread.SLEEP_OFFSET.

-   **Thread Safety:** The method executeSync, as its name implies, is designed to be called synchronously from the main server thread. The primary concurrency concern is the modification of TickingThread.SLEEP_OFFSET, which is read by the TickingThread itself.

    **Warning:** The safe operation of this command relies on the assumption that TickingThread.SLEEP_OFFSET is declared as **volatile** or is an atomic type. Without this, there is no guarantee that changes made by the command thread will be visible to the TickingThread, leading to unpredictable server loop behavior.

## API Surface
The public API is intended for framework consumption only. Direct invocation by developers is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Framework-invoked method that reads arguments from the context and mutates the global TickingThread.SLEEP_OFFSET. |

## Integration Patterns
This class is not designed to be integrated with via code. It is a user-facing utility invoked through the server console.

### Standard Usage
The command is executed by an administrator via the server console or an in-game chat interface with sufficient permissions.

**Get current offset in milliseconds:**
```
/sleep offset
```

**Set offset to 50 milliseconds:**
```
/sleep offset 50
```

**Get current offset as a percentage:**
```
/sleep offset --percent
```

**Set offset to 25 percent:**
```
/sleep offset 25 --percent
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SleepOffsetCommand()`. The command system manages instantiation and registration. Creating an instance manually has no effect.
-   **Manual Invocation:** Never call the `executeSync` method directly. Doing so bypasses the command system's argument parsing, permission handling, and lifecycle management.
-   **Unsafe Tuning:** Modifying the sleep offset to extreme values can severely impact server performance. A very low value may lead to 100% CPU usage, while a very high value will cause severe server lag and unresponsiveness. This command should be used with extreme caution.

## Data Pipeline
The data flow for this command is initiated by a user and results in a direct modification of a core server loop parameter.

> Flow:
> User Console Input -> Command Parser -> **SleepOffsetCommand** -> TickingThread.SLEEP_OFFSET (Static Field) -> Server Game Loop


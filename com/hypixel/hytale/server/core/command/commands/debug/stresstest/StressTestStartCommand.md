---
description: Architectural reference for StressTestStartCommand
---

# StressTestStartCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.stresstest
**Type:** Transient Command, Singleton Process Manager

## Definition
```java
// Signature
public class StressTestStartCommand extends AbstractAsyncWorldCommand {
```

## Architecture & Concepts

The StressTestStartCommand class serves as the user-facing entry point for initiating a server-wide stress test. While an instance of the command object is transient, its primary function is to configure and launch a singleton background process that systematically adds simulated players (Bots) to the server to measure performance degradation under load.

Architecturally, this class acts as a controller, not the test itself. The core test logic is managed by a set of static fields and scheduled tasks, ensuring that only one stress test can be active globally at any given time. This is enforced by a static, atomic state machine (`StressTestState`).

The fundamental mechanism is a feedback loop executed by a scheduled task:
1.  **Measure:** The system records world tick performance metrics over a defined interval.
2.  **Analyze:** Key percentile values (e.g., p95) of tick times are calculated.
3.  **Compare:** The calculated percentile is compared against a user-defined performance threshold.
4.  **Act:**
    *   If performance is within the acceptable threshold, a new Bot is created and added to the world. The loop continues.
    *   If performance exceeds the threshold, the test is automatically terminated, resources are cleaned up, and the server can be optionally shut down.

This design allows for a controlled, progressive load increase, automatically finding the server's performance ceiling for a given configuration.

## Lifecycle & Ownership

-   **Creation:** An instance of StressTestStartCommand is created by the server's command system whenever a privileged user executes the corresponding command, for example `/stresstest start`. It is not intended for manual instantiation.
-   **Scope:** The command object instance is short-lived and exists only for the duration of the `executeAsync` method call. However, the stress test *process* it initiates is long-running. This process, managed via static state and scheduled tasks, persists in the background across the entire server until it is explicitly stopped or meets its termination condition.
-   **Destruction:** The command object is eligible for garbage collection immediately after `executeAsync` completes. The background process it spawns must be terminated via the `StressTestStopCommand`, by exceeding its performance threshold, or by server shutdown. The static `stop` method is responsible for canceling scheduled tasks, unregistering event listeners, and clearing all static state to prevent resource leaks.

## Internal State & Concurrency

-   **State:** The StressTestStartCommand instance is stateless. All state related to an active stress test is stored in static fields within the class. This includes the global state machine (`STATE`), the collection of active bots (`BOTS`), handles to scheduled tasks (`STRESS_TEST_BOT_TASK`), and configuration details like file paths for metric dumps. This static state is highly mutable during a test run.

-   **Thread Safety:** The system is designed to be thread-safe. Concurrency is managed through several mechanisms:
    *   **State Machine:** The `STATE` field is an `AtomicReference`, ensuring that transitions between `NOT_RUNNING`, `RUNNING`, and `STOPPING` are atomic operations. This acts as the primary lock to prevent multiple tests from running simultaneously.
    *   **Bot Collection:** The `BOTS` list is a `Collections.synchronizedList`, allowing for safe additions and removals from multiple threads, although in practice modifications are serialized within the main task loop.
    *   **Executor Service:** The core feedback loop is submitted to the global `HytaleServer.SCHEDULED_EXECUTOR`. This offloads the test logic from the command execution thread and ensures that the periodic analysis and bot creation occur in a controlled, scheduled manner.

## API Surface

The primary interaction with this class is through the command system, which invokes the inherited `executeAsync` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context, world) | CompletableFuture<Void> | O(1) | Parses command arguments, validates them, and initiates the singleton stress test process. Returns immediately after starting the background task. Throws if a test is already running. |

## Integration Patterns

### Standard Usage

This class is designed to be used as a server command from the console or by a privileged player. It accepts numerous arguments to configure the test parameters.

```
# Example: Start a test that adds a bot every 10 seconds,
# stopping when the 95th percentile tick time exceeds 30ms.
# The server will shut down upon completion.

/stresstest start interval=10 threshold=30 percentile=0.95 --shutdown
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance via `new StressTestStartCommand()`. The class is useless outside the context of the command system, which provides the necessary `CommandContext` and `World` arguments.
-   **External State Manipulation:** Directly modifying the static fields such as `StressTestStartCommand.STATE` or `StressTestStartCommand.BOTS` from external code is extremely dangerous. This will break the internal state machine, bypass safety checks, and likely lead to severe resource leaks or server instability.
-   **Bypassing the State Lock:** Any attempt to circumvent the `STATE.compareAndSet` check to run multiple tests will fail and cause unpredictable behavior, as the static resources (event listeners, scheduled tasks) are not designed for concurrent ownership.

## Data Pipeline

The component operates on a control flow rather than a traditional data pipeline. It orchestrates a feedback loop based on server performance metrics.

> Flow:
> User Command -> Command System -> **StressTestStartCommand.executeAsync** -> Static `start` method -> HytaleServer.SCHEDULED_EXECUTOR -> **Feedback Loop**
>
> **Feedback Loop Internals:**
> 1.  World Tick Metric -> `HistoricMetric`
> 2.  `HistoricMetric` -> Percentile Calculation
> 3.  Result -> Write to CSV Log
> 4.  Result -> Compare with Threshold
> 5.  IF (under threshold) -> Add New Bot
> 6.  IF (over threshold) -> Trigger Static `stop` method -> Resource Cleanup


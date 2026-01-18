---
description: Architectural reference for SleepTestCommand
---

# SleepTestCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.sleep
**Type:** Transient

## Definition
```java
// Signature
public class SleepTestCommand extends CommandBase {
```

## Architecture & Concepts
The SleepTestCommand is a diagnostic tool integrated into the server's command system. Its primary function is to measure the precision and overhead of the Java Virtual Machine's `Thread.sleep` method, which is a fundamental component for any system that relies on timed waits or pauses.

Architecturally, this command serves as a non-invasive performance benchmark. It extends the CommandBase class, inheriting the standard structure for command registration, argument parsing, and execution context handling.

The most critical design choice is its asynchronous execution model. The core logic, which involves a potentially long-running loop of sleep calls, is intentionally offloaded from the main server thread using `CompletableFuture.runAsync`. This prevents the command from blocking the server's primary tick loop or command processing queue, which would otherwise result in catastrophic server-wide freezes. This class is a model for how to correctly implement long-running or blocking diagnostic tasks within the server environment.

## Lifecycle & Ownership
- **Creation:** An instance of SleepTestCommand is created by the server's command registration service during the server bootstrap sequence. It is discovered, instantiated, and added to a central command registry.
- **Scope:** The command object is a stateless handler. A single instance exists for the entire duration of the server's runtime, held in memory by the command registry.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. The fields `sleepArg` and `countArg` are final and define the command's argument structure, not its runtime state. All state related to a specific execution (e.g., metrics, loop counters) is created and destroyed within the local scope of the `executeSync` method's asynchronous lambda.

- **Thread Safety:** The object itself is inherently thread-safe due to its stateless nature. The `executeSync` method is invoked by the command system, but it immediately dispatches the workload to a background thread from the common ForkJoinPool. All interactions with the CommandContext, such as `sendMessage`, are designed to be thread-safe, allowing the background task to safely report its progress and results back to the command's originator.

## API Surface
The public contract of this class is fulfilled by its registration as a server command, not through direct programmatic invocation. The primary entry point is the `executeSync` method, which is part of the protected contract with CommandBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Initiates the sleep test. The method itself returns in O(1), but the asynchronous task it launches runs in O(N) where N is the `count` argument. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked by a server administrator or automated system via the server console or in-game chat.

```java
// This class is not a service and should not be retrieved from a context.
// It is invoked automatically by the command system.

// Example of administrative usage from the server console:
// > sleeptest test sleep:20 count:500
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SleepTestCommand()`. The command system manages its lifecycle. A manually created instance will not be registered and is non-functional.
- **Synchronous Execution:** Re-implementing this logic without offloading the `Thread.sleep` loop to a background thread is a severe anti-pattern. Doing so would block the calling thread, likely leading to a server freeze or timeout.

## Data Pipeline
The data flow for this command begins with user input and ends with formatted output sent back to the user. The core processing is intentionally decoupled from the main server threads.

> Flow:
> Server Console Input -> Command Parser -> **SleepTestCommand.executeSync** -> CompletableFuture -> Background Worker Thread (ForkJoinPool) -> Metric Calculation -> CommandContext.sendMessage -> Server Console Output


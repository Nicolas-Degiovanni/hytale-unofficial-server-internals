---
description: Architectural reference for NPCBenchmarkCommand
---

# NPCBenchmarkCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCBenchmarkCommand extends CommandBase {
```

## Architecture & Concepts
The NPCBenchmarkCommand class provides a server-side, user-facing entry point for triggering performance analysis of the Non-Player Character (NPC) subsystem. It functions as a specialized command within the server's command framework, inheriting from the CommandBase class.

Architecturally, this class is a thin translation layer. It bridges human-readable chat commands with the powerful, but less accessible, internal benchmarking capabilities of the NPCPlugin. Its primary responsibilities are:
1.  **Argument Parsing:** Defining and parsing command-line flags and arguments, such as the benchmark type (--roles, --sensorsupport) and duration (seconds).
2.  **Delegation:** Invoking the appropriate benchmark method on the central NPCPlugin singleton.
3.  **Asynchronous Result Handling:** Providing a callback lambda to the NPCPlugin to process benchmark results once the measurement period is complete.
4.  **Reporting:** Formatting the raw performance data into a structured, human-readable report and delivering it to both the command issuer and the server log.

This command is not intended for game logic but serves as a critical diagnostic and profiling tool for developers and server administrators.

### Lifecycle & Ownership
-   **Creation:** An instance of NPCBenchmarkCommand is created by the server's command registration system when the NPCPlugin is loaded. The plugin is responsible for registering this command, making it available for execution.
-   **Scope:** The object instance persists for the entire server session, held within the central command registry. It does not maintain state between executions.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the NPCPlugin is unloaded or the server shuts down, at which point the command is de-registered.

## Internal State & Concurrency
-   **State:** The class instance itself is effectively **stateless** between invocations. Fields like roleArg and secondsArg are configuration objects that define the command's arguments; they are initialized in the constructor and are treated as **immutable** thereafter. All execution-specific data is provided via the CommandContext object passed into the executeSync method.

-   **Thread Safety:** This class is **not thread-safe** and is not designed to be. The command system guarantees that the executeSync method is always invoked on the main server thread. This synchronous execution model prevents race conditions within the command's logic.

    **WARNING:** The command initiates an asynchronous benchmark task within the NPCPlugin. The callback lambda provided for result handling will also be executed on the main server thread at a later time, ensuring that interactions with the CommandContext and server logging are safe.

## API Surface
The public contract is exclusively defined by its role as a CommandBase implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(1) | The entry point called by the command system. Parses arguments and triggers an asynchronous benchmark in the NPCPlugin. The method returns immediately; the benchmark runs in the background. Throws exceptions on invalid argument parsing. |

## Integration Patterns

### Standard Usage
This class is designed to be used by server administrators or developers via the in-game chat or server console. It is not intended to be invoked programmatically from other code.

The command initiates a benchmark that runs for a specified duration. Upon completion, a detailed report is printed to the server log and a confirmation message is sent to the user.

**Example Console Invocation:**
```
/npc benchmark --roles --seconds 60
```
This command starts a 60-second performance benchmark of the NPC role system.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance using `new NPCBenchmarkCommand()`. The command must be registered with and managed by the server's command system to function correctly.
-   **Direct Method Invocation:** Never call the `executeSync` method directly. Doing so bypasses the entire command processing pipeline, including permission checks, argument parsing, and context creation, leading to unpredictable behavior and likely NullPointerExceptions.
-   **Concurrent Execution:** Do not attempt to start a new benchmark while another is already running. The NPCPlugin may reject the request, and the command's feedback mechanism is not designed to handle multiple concurrent reports. The `success` flag check provides basic protection against this.

## Data Pipeline
The flow of data and control for this command is linear and event-driven, involving user input, the command system, and the NPC subsystem.

> Flow:
> User Command Input -> Server Command Parser -> **NPCBenchmarkCommand.executeSync** -> NPCPlugin.start...Benchmark -> (Async Benchmark Execution) -> NPCPlugin invokes Callback -> **NPCBenchmarkCommand formats results** -> Server Logger & User Chat Message


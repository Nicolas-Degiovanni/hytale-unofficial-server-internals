---
description: Architectural reference for WorldGenBenchmarkCommand
---

# WorldGenBenchmarkCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.worldgen
**Type:** Managed Component

## Definition
```java
// Signature
public class WorldGenBenchmarkCommand extends CommandBase {
```

## Architecture & Concepts
The WorldGenBenchmarkCommand is a server-side diagnostic tool designed to measure the performance of a world's terrain generation algorithm. It operates within the server's Command System, providing an in-game interface for administrators to trigger performance analysis on-demand.

Architecturally, this class serves as a lightweight entry point that orchestrates a complex, long-running, and resource-intensive background task. Its primary design goal is to isolate the performance-heavy benchmarking process from the main server thread, ensuring that server playability is not impacted during a benchmark run.

This is achieved through a two-part execution model:
1.  **Synchronous Initiation:** The initial command execution validates arguments and checks preconditions on the main server thread.
2.  **Asynchronous Execution:** The core world generation and data aggregation logic is dispatched to a dedicated worker thread.

A critical architectural feature is the use of a static AtomicBoolean, which acts as a server-wide mutex. This ensures that only one benchmark can be active at any given time across the entire server instance, preventing resource contention and data corruption. The command also relies on a specific interface, IBenchmarkableWorldGen, tightly coupling its functionality to world generators that explicitly support performance metric collection.

## Lifecycle & Ownership
-   **Creation:** An instance of WorldGenBenchmarkCommand is created by the server's Command System during the command registration phase, typically at server startup. It is not intended for manual instantiation.
-   **Scope:** The command object itself persists for the entire server session, held as a reference by the Command System. However, the core logic and the worker thread it spawns are ephemeral, existing only for the duration of a single benchmark execution.
-   **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the server shuts down or the command is programmatically unregistered. The associated worker thread terminates upon completion of its task or in the case of an unhandled exception.

## Internal State & Concurrency
-   **State:** The WorldGenBenchmarkCommand instance is effectively stateless. The most significant state is the static `IS_RUNNING` AtomicBoolean field. This is a mutable, globally shared flag that represents the server-wide lock for the benchmarking process.
-   **Thread Safety:** This class employs a sophisticated threading model to maintain server stability.
    -   The entry point, executeSync, is invoked on the main server thread.
    -   A global, non-blocking lock is acquired using the `IS_RUNNING` atomic. This is a thread-safe mechanism to prevent concurrent benchmark executions.
    -   All chunk generation, data processing, and file I/O are offloaded to a newly spawned worker thread named "WorldGenBenchmarkCommand".
    -   **WARNING:** Communication from the worker thread back to the game world (e.g., sending progress messages to players) is safely marshaled back to the main server thread via the `world.execute` method. Direct interaction with game state from the worker thread would lead to severe concurrency violations.

## API Surface
The primary public contract is its behavior as a registered command, not a traditional programmatic API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Initiates the benchmark process. This method returns almost immediately, dispatching the full workload to a background thread. The true complexity is O(N) where N is the number of chunks in the selected region, but this work is not performed on the calling thread. |

## Integration Patterns

### Standard Usage
This component is designed to be used exclusively as an in-game or console command. The command syntax defines its public interface.

```
# Example command to benchmark a 10x10 chunk area
# around the coordinates <100, 100> and <260, 260>.
/world worldgen benchmark 100 100 260 260
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new WorldGenBenchmarkCommand()`. The object has no utility outside of the command registration system and will not function correctly.
-   **Bypassing The Lock:** Do not use reflection or other means to modify the `IS_RUNNING` state variable. Doing so will break the server-wide mutex, allowing multiple benchmarks to run concurrently, which can lead to server crashes, resource exhaustion, and invalid report data.
-   **Assuming Generator Compatibility:** Do not assume any world generator can be benchmarked. The command will fail with a clear error if the target world's generator does not implement the IBenchmarkableWorldGen interface.

## Data Pipeline
The flow of data and control is linear but crosses multiple thread boundaries.

> Flow:
> User Command Input -> Command System -> **WorldGenBenchmarkCommand.executeSync** (Main Thread)
> -> Spawns Worker Thread
> -> Worker Thread loops, calling `IWorldGen.generate()` for each chunk
> -> `IBenchmarkableWorldGen` implementation collects performance timings
> -> Worker Thread aggregates timings into a final report string
> -> Report String is written to a `.txt` file in the `quantification` directory (Blocking I/O on Worker Thread)
> -> Progress and completion messages are sent via `world.execute()` back to the Main Thread for display to the user.


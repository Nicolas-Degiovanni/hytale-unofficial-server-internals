---
description: Architectural reference for DumpUtil
---

# DumpUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class DumpUtil {
```

## Architecture & Concepts
The DumpUtil class is a high-level diagnostic and reporting facility for the Hytale server. It is not part of the real-time game simulation loop; instead, it functions as an on-demand service for capturing a comprehensive snapshot of the entire server's state. Its primary role is to serve as a centralized aggregator for debugging, crash analysis, and performance profiling.

The system is designed to query a wide array of disparate server subsystems, including the Universe, individual Worlds, Player entities, the Entity-Component-System (ECS) stores, the plugin manager, and the underlying Java Virtual Machine. It then consolidates this information into two distinct output formats:

1.  A human-readable text file (`dump.txt`), which includes detailed summaries, performance graphs, and full thread dumps for manual inspection by developers.
2.  A machine-readable JSON file (`dump.json`), which provides a structured representation of server metrics suitable for automated analysis and monitoring systems.

A critical architectural feature is its resilient and concurrent data collection strategy. To avoid deadlocking the server during a dump operation—especially in a partially hung or failing state—it parallelizes data gathering across Worlds using `CompletableFuture`. Each collection task is constrained by a strict timeout, ensuring that the dump process itself can complete even if a specific subsystem is unresponsive.

### Lifecycle & Ownership
- **Creation:** As a static utility class, DumpUtil is never instantiated. Its methods are invoked directly from the class definition.
- **Scope:** The class is stateless. Its operational scope is confined to the duration of a single static method invocation. It does not maintain any state between calls.
- **Destruction:** The class is part of the server's core classpath and is unloaded only when the Java Virtual Machine shuts down. No manual cleanup is required.

## Internal State & Concurrency
- **State:** DumpUtil is fundamentally **stateless**. It does not contain any instance or static fields that store data between invocations. All information is gathered, processed, and written to an output stream within the context of a single method call.

- **Thread Safety:** The public API is designed to be invoked from a single thread, such as a command handler or a JVM shutdown hook. However, its internal implementation is highly concurrent. The `collectPlayerComponentMetrics` and `collectPlayerTextData` methods dispatch asynchronous tasks to each World's dedicated thread pool. Results are safely collected into concurrent data structures like `ConcurrentHashMap`. This parallel execution model is essential for minimizing the performance impact of a full server dump.

**WARNING:** While the data collection is parallelized, the initial orchestration and final file I/O occur on the calling thread. Invoking a dump from a time-sensitive thread like the main server tick loop is extremely hazardous and can lead to significant stalls.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| dumpToJson() | Path | O(N) | Orchestrates the collection of server-wide metrics and state, serializing the result into a BSON document and writing it as a JSON file. Throws IOException on file system errors. |
| dump(crash, printToConsole) | Path | O(N) | Generates a comprehensive, human-readable text report of the entire server state, including JVM metrics, thread dumps, and game-specific data. |
| hexDump(buf) | String | O(M) | Converts the readable bytes of a Netty ByteBuf into a hexadecimal string for debugging network packets or binary data. |
| createDumpPath(crash, ext) | Path | O(1) | Generates a unique, timestamped file path within the 'dumps' directory for a new report, avoiding name collisions. |

## Integration Patterns

### Standard Usage
DumpUtil is typically invoked from server administration commands or, more critically, from a global exception handler or shutdown hook to capture the server's state upon failure.

```java
// Example: Integrating into a server crash handler
try {
    // ... server logic fails
} catch (Throwable catastrophicError) {
    System.err.println("A catastrophic error occurred. Generating crash report...");
    try {
        // Generate a text dump marked as a crash and print it to the console.
        Path reportPath = DumpUtil.dump(true, true);
        System.err.println("Crash report successfully saved to: " + reportPath);
    } catch (Exception dumpError) {
        System.err.println("FATAL: Failed to generate crash report.");
        dumpError.printStackTrace();
    }
    // Proceed with server shutdown
}
```

### Anti-Patterns (Do NOT do this)
- **Frequent Invocation:** The `dump` and `dumpToJson` methods are heavyweight, resource-intensive operations. Calling them in a loop or on a frequent, scheduled basis will severely degrade server performance and cause significant I/O load. They are intended for exceptional circumstances only.
- **Main Thread Execution:** Do not invoke `dump` or `dumpToJson` directly from the main server game loop. The potential for thread contention and I/O blocking can cause catastrophic tick-rate drops and server-wide freezes. If a dump must be triggered by game logic, it should be dispatched to a separate, low-priority administrative thread.

## Data Pipeline
The data flow for DumpUtil is a classic "gather" or "fan-in" pattern. A single entry point triggers a parallel fan-out to numerous system components, followed by an aggregation phase and final serialization to disk.

> **Human-Readable Dump Flow:**
> Trigger (Crash Handler / Command) -> **DumpUtil.dump()** -> Parallel Data Collection [JVM ManagementFactory, Universe, HytaleServer, MetricsRegistry] -> Data Aggregation & Formatting -> PrintWriter -> FileOutputStream -> `dump.txt`

> **JSON Dump Flow:**
> Trigger (API / Command) -> **DumpUtil.dumpToJson()** -> Parallel Data Collection [Universe, HytaleServer, MetricsRegistry] -> BSON Document Assembly -> JSON Serialization -> Files.writeString -> `dump.json`


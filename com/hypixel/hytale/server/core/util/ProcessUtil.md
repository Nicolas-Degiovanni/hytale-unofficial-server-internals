---
description: Architectural reference for ProcessUtil
---

# ProcessUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public final class ProcessUtil {
    // Note: A private constructor is implied to prevent instantiation.
}
```

## Architecture & Concepts
ProcessUtil is a low-level, stateless utility class that provides a critical bridge between the Hytale server application and the underlying host operating system. Its sole function is to abstract process state queries, allowing higher-level server management components to monitor the health and existence of external or child processes without dealing with platform-specific APIs.

This component resides outside the core game loop and simulation logic. It is typically leveraged by infrastructure services, such as a **World Server Manager** or a **Daemon Supervisor**, which are responsible for launching, monitoring, and restarting dependent server instances. By providing a clean, static interface, it decouples the server's orchestration logic from the complexities of OS-level process management.

## Lifecycle & Ownership
As a static utility class, ProcessUtil has no instances and therefore no traditional object lifecycle.

- **Creation:** The class is loaded into the JVM by a ClassLoader when one of its static methods is first invoked. It is never instantiated.
- **Scope:** The class and its static methods are available globally for the entire lifetime of the server's JVM process.
- **Destruction:** The class is unloaded from memory only when the JVM itself shuts down. No explicit cleanup is necessary.

## Internal State & Concurrency
- **State:** ProcessUtil is completely **stateless**. It maintains no internal fields, caches, or configuration. Each method call is an independent, atomic operation.
- **Thread Safety:** This class is inherently **thread-safe**. Its methods are pure functions that do not modify any shared state. Multiple threads can safely and concurrently call its methods without locks or synchronization primitives. The underlying call to the Java `ProcessHandle` API is also guaranteed to be thread-safe.

## API Surface
The public contract is minimal, consisting of a single static query method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isProcessRunning(int pid) | boolean | O(1) | Queries the OS to check for the existence of a process with the given Process ID (PID). Returns true if the process is found, false otherwise. |

## Integration Patterns

### Standard Usage
This utility should be invoked directly by any system component that needs to verify the status of an external process it manages.

```java
// A supervisor service checks if a managed world server is still alive.
int managedProcessId = getManagedWorldPid();

if (!ProcessUtil.isProcessRunning(managedProcessId)) {
    // The process has terminated unexpectedly.
    // Trigger a restart or alert an administrator.
    log.error("Monitored process " + managedProcessId + " has terminated.");
    restartProcess(managedProcessId);
}
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Never attempt to create an instance with `new ProcessUtil()`. This is a static utility and should have a private constructor to prevent this programmatically.
- **High-Frequency Polling:** Avoid calling `isProcessRunning` in a tight, high-frequency loop (e.g., many times per second). While the operation is fast, excessive polling places unnecessary load on the operating system scheduler. For health checks, a scheduled task running at a reasonable interval (e.g., every 1-5 seconds) is the correct pattern.

## Data Pipeline
The data flow for this component is a simple, synchronous query to the operating system.

> Flow:
> Server Orchestrator -> **ProcessUtil.isProcessRunning(pid)** -> Java `ProcessHandle` API -> Operating System Process Table -> Boolean Response


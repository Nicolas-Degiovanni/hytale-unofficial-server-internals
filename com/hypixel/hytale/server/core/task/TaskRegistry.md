---
description: Architectural reference for TaskRegistry
---

# TaskRegistry

**Package:** com.hypixel.hytale.server.core.task
**Type:** Singleton

## Definition
```java
// Signature
public class TaskRegistry extends Registry<TaskRegistration> {
```

## Architecture & Concepts
The TaskRegistry is a specialized, server-side implementation of the generic Registry pattern. Its primary architectural role is to serve as a centralized manager for all long-running or scheduled asynchronous operations within the server core. It provides a unified system for tracking, monitoring, and controlling the lifecycle of background work, which is represented by standard Java concurrency objects like CompletableFuture and ScheduledFuture.

By extending the base Registry class, TaskRegistry inherits the core mechanics of collection management, preconditions, and registration callbacks. Its specialization provides type-safe, intention-revealing methods for handling asynchronous tasks, abstracting away the underlying storage mechanism. This centralization is critical for ensuring server stability, enabling graceful shutdowns, and preventing resource leaks from unmanaged, "fire-and-forget" background processes. It is a foundational component of the server's concurrency and task execution model.

### Lifecycle & Ownership
- **Creation:** The TaskRegistry is instantiated once during the server's main bootstrap sequence. Its creation is managed by a central service locator or dependency injection container, which supplies the necessary preconditions and callback lists required by its constructor.
- **Scope:** Session-scoped. A single instance of TaskRegistry persists for the entire operational lifetime of the server process.
- **Destruction:** The registry is decommissioned during the server shutdown sequence. A system-level process is responsible for iterating through all registered tasks and signaling cancellation to ensure a clean and orderly shutdown.

## Internal State & Concurrency
- **State:** The state of the TaskRegistry is highly mutable, as its primary function is to maintain a dynamic collection of active TaskRegistration objects. The underlying data structure, managed by the parent Registry class, is designed to grow and shrink as tasks are added and completed.
- **Thread Safety:** This class is thread-safe and designed for high-concurrency environments. The core registration logic, inherited from the parent Registry, almost certainly employs concurrent collections and synchronization primitives to safely handle task registrations from multiple server threads simultaneously, such as network I/O threads, world tick loops, and AI processors.

## API Surface
The public API is intentionally minimal, focusing exclusively on the act of registration. Management and introspection are likely handled through the inherited methods of the parent Registry class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerTask(CompletableFuture task) | TaskRegistration | O(1) | Registers an active CompletableFuture. Returns a registration handle for potential future interaction. |
| registerTask(ScheduledFuture task) | TaskRegistration | O(1) | Registers a task scheduled for future execution. Returns a registration handle. |

## Integration Patterns

### Standard Usage
The TaskRegistry should be retrieved from the server's central service context. Any subsystem that spawns a long-running asynchronous operation must register the resulting future with this registry.

```java
// Example: Submitting and registering a database query task
ExecutorService databaseExecutor = ...;
TaskRegistry taskRegistry = serverContext.getService(TaskRegistry.class);

CompletableFuture<Void> dbQuery = CompletableFuture.runAsync(() -> {
    // Perform long-running database operation
}, databaseExecutor);

// Register the task to ensure it is tracked by the server
TaskRegistration handle = taskRegistry.registerTask(dbQuery);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new TaskRegistry()`. The constructor requires internal engine parameters (preconditions, callbacks) that are configured only at server startup. Always retrieve the shared instance from the application context.
- **Unregistered Futures:** Spawning a background task via an ExecutorService and failing to register the returned future is a critical error. This creates an unmanaged task that the server cannot track, which can prevent clean shutdowns and lead to resource leaks.

## Data Pipeline
TaskRegistry does not process a flow of data. Instead, it manages a control flow for asynchronous operations.

> Flow:
> System Event (e.g., Player Login) -> Task Submitted to Executor -> `CompletableFuture` or `ScheduledFuture` created -> **`TaskRegistry.registerTask()`** -> `TaskRegistration` handle returned -> (On Shutdown) Server iterates registry -> `Future.cancel()` invoked on all tasks


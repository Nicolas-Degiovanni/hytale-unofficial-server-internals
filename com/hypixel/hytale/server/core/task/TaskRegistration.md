---
description: Architectural reference for TaskRegistration
---

# TaskRegistration

**Package:** com.hypixel.hytale.server.core.task
**Type:** Transient

## Definition
```java
// Signature
public class TaskRegistration extends Registration {
```

## Architecture & Concepts
The TaskRegistration class is a specialized handle that represents a scheduled, asynchronous operation within the server's core task execution framework. It acts as a concrete implementation of the abstract `Registration` concept, binding a generic registration lifecycle to a standard Java Concurrency `Future`.

This class serves as the primary bridge between the server's high-level task scheduling API and the low-level Java `ExecutorService` that performs the work. When a system submits a task (e.g., a repeating world save, a temporary player effect), the scheduler returns a TaskRegistration instance. This object encapsulates the underlying `Future`, providing the caller with a token that can be used to manage the task's lifecycle, most critically for cancellation.

By extending `Registration`, it inherits a standardized contract for enabling, disabling, and unregistering, which is then wired directly to the `cancel` method of the encapsulated `Future`. This design decouples the task-submitting code from the implementation details of the concurrency framework.

### Lifecycle & Ownership
- **Creation:** A TaskRegistration is instantiated exclusively by a task scheduler service when a new task is submitted and accepted by an underlying `ExecutorService`. The scheduler creates the `Future` and immediately wraps it within a new TaskRegistration object before returning it to the caller.
- **Scope:** The object's lifetime is tied to the scheduled task it represents. It is intended to be held by the system that initiated the task for as long as that task may need to be managed or cancelled.
- **Destruction:** The object becomes eligible for garbage collection once all references to it are released. This typically occurs after its `unregister` method is called (which cancels the underlying task) or after the task completes naturally and the handle is no longer needed.

## Internal State & Concurrency
- **State:** The internal state is minimal and immutable. It consists of a single `final` field, `task`, which holds a reference to a `Future`. The state of the underlying `Future` object is, by nature, mutable (e.g., it can transition from running to done or cancelled), but the TaskRegistration's reference to it does not change post-construction.
- **Thread Safety:** This class is thread-safe. The internal reference is final, and the public `getTask` method is a simple, non-blocking read. All interactions with the underlying `Future` object, including the `cancel` operation invoked by the parent class's `unregister` method, are inherently thread-safe as per the Java Concurrency API contract. It is safe to share and access a TaskRegistration instance across multiple threads.

## API Surface
The public API is focused on providing access to the underlying task handle. The primary lifecycle management methods (`isEnabled`, `unregister`) are inherited from the parent `Registration` class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TaskRegistration(Future) | constructor | O(1) | Primary constructor. Binds a `Future` to a registration handle. The unregister action is hardwired to `task.cancel(false)`. |
| TaskRegistration(TaskRegistration, ...) | constructor | O(1) | Decorator constructor. Creates a new registration with custom lifecycle logic that still wraps the same underlying `Future`. |
| getTask() | Future<?> | O(1) | Retrieves the raw `Future` object associated with this registration. Use with caution. |

## Integration Patterns

### Standard Usage
The intended pattern is to capture the TaskRegistration object upon task submission and use its inherited `unregister` method to terminate the task when it is no longer needed.

```java
// A scheduler service is retrieved from the context
TaskScheduler scheduler = context.getService(TaskScheduler.class);

// A task is submitted, and the handle is retained
TaskRegistration repeatingSaveTask = scheduler.runRepeating(() -> {
    world.save();
}, 20 * 60);

// Later, during server shutdown, the task is cancelled
// by invoking the method from the parent Registration class.
repeatingSaveTask.unregister();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create this object directly with `new TaskRegistration()`. The `Future` it contains must originate from a managed `ExecutorService` to be meaningful. Always acquire instances from a scheduler service.
- **Misusing getTask:** Avoid calling `getTask().cancel()` directly. The `unregister()` method provides the intended level of abstraction and may include additional cleanup logic in the future. Bypassing it breaks the registration contract.
- **Ignoring the Handle:** For any task that is not a "fire-and-forget" operation, discarding the returned TaskRegistration is an error. Without the handle, there is no standard way to cancel a long-running or repeating task, which can lead to resource leaks or unwanted behavior.

## Data Pipeline
TaskRegistration is not part of a data processing pipeline. Instead, it functions within a control flow for managing asynchronous operations.

> Control Flow:
> System Code -> `TaskScheduler.submit(Runnable)` -> `ExecutorService` creates `Future` -> **`new TaskRegistration(Future)`** is created and returned -> System Code holds handle -> `registration.unregister()` is called -> `Future.cancel()` is invoked on the underlying task.


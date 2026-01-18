---
description: Architectural reference for ShutdownReason
---

# ShutdownReason

**Package:** com.hypixel.hytale.server.core
**Type:** Value Object

## Definition
```java
// Signature
public class ShutdownReason {
```

## Architecture & Concepts

The ShutdownReason class is a foundational component of the server's lifecycle and error handling architecture. It implements a type-safe enumeration pattern to represent the explicit cause of a server termination. Its primary design goal is to decouple the *detection* of a fatal error from the *execution* of the shutdown sequence.

This class acts as a standardized data contract, providing a structured way for disparate subsystems—such as the authentication service, world generator, or plugin loader—to signal a terminal state to the main server process. It combines a machine-readable integer exit code, suitable for scripting and automated monitoring, with an optional human-readable message for logging and diagnostics.

The provision of predefined static constants (e.g., SHUTDOWN, CRASH) establishes a vocabulary for common termination scenarios, while the public constructors allow for the creation of context-specific reasons for more nuanced error conditions.

### Lifecycle & Ownership
- **Creation:** Instances are created under two distinct circumstances:
    1. **Statically:** A set of common, well-known reasons are instantiated as public static final fields during class loading (e.g., ShutdownReason.SIGINT).
    2. **Dynamically:** Any server subsystem can instantiate a new ShutdownReason on-demand when it encounters a condition that requires server termination.
- **Scope:** Short-lived and transactional. An instance typically exists only for the duration of the shutdown signaling process. It is created, passed to the core server shutdown handler, and then becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. As these are simple value objects with no external resources, no explicit cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are final. Methods that appear to modify the object, such as withMessage, follow functional programming principles by returning a new, modified instance rather than altering the state of the original. This design guarantees that a ShutdownReason object is a stable and predictable value once created.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, a ShutdownReason instance can be safely shared, passed, and read across multiple threads without any form of external synchronization or locking. This is critical in a multi-threaded server environment where a fatal error could be detected on any number of worker threads.

## API Surface

The public API is minimal, focusing exclusively on data access and non-mutating transformations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getExitCode() | int | O(1) | Returns the integer exit code intended for the operating system process. |
| getMessage() | String | O(1) | Returns the human-readable diagnostic message, which may be null. |
| withMessage(String) | ShutdownReason | O(1) | **Returns a new instance.** Clones the current object, replacing its message. The original object is unaffected. |

## Integration Patterns

### Standard Usage

A subsystem should create or select a ShutdownReason and pass it to the central server's shutdown method. The withMessage method is the preferred way to add context to a predefined reason.

```java
// Example: A plugin loader detects a critical failure
try {
    pluginManager.loadAll();
} catch (MissingDependencyException e) {
    // Use a predefined reason but add specific context for the logs
    ShutdownReason reason = ShutdownReason.MISSING_REQUIRED_PLUGIN
        .withMessage("Plugin 'CoreServices' failed to load: " + e.getMessage());

    // Signal the server to terminate
    server.shutdown(reason);
}
```

### Anti-Patterns (Do NOT do this)
- **Re-implementing Constants:** Do not create `new ShutdownReason(1)` when `ShutdownReason.CRASH` is available. Using the predefined static constants improves code readability and maintains consistency across the codebase.
- **Stateful Logic:** Do not subclass ShutdownReason to add complex logic or state. It is designed to be a simple, inert data carrier.
- **Ignoring Messages:** When creating a custom shutdown reason for a specific, non-standard error, always provide a descriptive message. A custom exit code without a message is a diagnostic dead-end.

## Data Pipeline

ShutdownReason is a data source that initiates the server termination pipeline. It does not process data itself; rather, it *is* the data that flows through the shutdown sequence.

> Flow:
> Error Condition in Subsystem -> **Instantiation of ShutdownReason** -> ServerCore.shutdown(reason) -> Global Shutdown Event -> System.exit(reason.getExitCode()) -> Process Termination


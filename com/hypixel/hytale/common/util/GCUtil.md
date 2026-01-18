---
description: Architectural reference for GCUtil
---

# GCUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class GCUtil {
```

## Architecture & Concepts
GCUtil is a low-level static utility that provides a direct interface to the Java Virtual Machine's garbage collection monitoring system. It acts as a simplified facade over the Java Management Extensions (JMX) API, specifically targeting GC notifications.

Its primary role within the engine is to enable performance monitoring and diagnostic systems to subscribe to GC events without needing to interact with the verbose and complex JMX `NotificationEmitter` and `CompositeData` APIs. By abstracting this interaction, it provides a clean, Hytale-idiomatic entry point for observing JVM memory management behavior. This component is crucial for debugging memory-related performance issues and for powering in-game developer diagnostic tools.

## Lifecycle & Ownership
- **Creation:** As a static utility class, GCUtil is never instantiated. Its methods are invoked statically. The internal listeners it creates are instantiated on-demand when the `register` method is called.
- **Scope:** The notification listeners registered by this utility are attached to the JVM's `GarbageCollectorMXBean` instances. These listeners persist for the entire lifetime of the application. There is no provided mechanism for unregistering them.
- **Destruction:** Listeners are only cleaned up when the JVM shuts down. This implies that any consumer registered is intended to be permanent.

## Internal State & Concurrency
- **State:** This class is entirely stateless, containing no member fields. It does not cache any data.
- **Thread Safety:** The `register` method is safe to call from any thread. However, the provided `Consumer` callback is **not** invoked on the calling thread. The JVM's JMX notification system will execute the consumer on a dedicated, internal JMX dispatcher thread.

**WARNING:** All `Consumer` implementations passed to `register` **must be thread-safe**. Any state they modify must be protected by appropriate synchronization mechanisms (e.g., locks, atomic variables) to prevent race conditions with the main game loop or other engine threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(Consumer consumer) | static void | O(N) | Registers a consumer to receive notifications for all GC events. N is the number of garbage collectors in the JVM, which is a small constant. |

## Integration Patterns

### Standard Usage
This utility should be used by engine services during their initialization phase to set up permanent monitoring of GC activity. A common use case is a performance metrics service that logs GC pauses.

```java
// In a performance monitoring service's initialization method
GCUtil.register(info -> {
    // This code executes on a JMX thread, not the main thread.
    // Ensure logging or state updates are thread-safe.
    String gcName = info.getGcName();
    long duration = info.getGcInfo().getDuration();
    LOGGER.info("GC Event: " + gcName + " took " + duration + "ms.");
});
```

### Anti-Patterns (Do NOT do this)
- **Non-Thread-Safe Consumers:** Passing a consumer that directly modifies non-thread-safe game state (e.g., a UI component, a game world object) will lead to memory visibility issues, data corruption, and crashes. All shared data must be accessed via thread-safe mechanisms.
- **Multiple Registrations:** Calling `register` repeatedly from the same system will install duplicate listeners, causing the consumer logic to execute multiple times for each single GC event. Listeners should be registered once during system initialization.
- **Assuming Main Thread Execution:** Do not perform operations in the consumer that are required to run on the main game thread without dispatching them back to that thread via a scheduler or event queue.

## Data Pipeline
The data flow originates deep within the JVM and is propagated outwards to engine-level systems via this utility.

> Flow:
> JVM Garbage Collector Run -> JMX Notification Emitter -> **GCUtil Listener** -> Registered Consumer Callback -> Engine System (e.g., Performance Logger)


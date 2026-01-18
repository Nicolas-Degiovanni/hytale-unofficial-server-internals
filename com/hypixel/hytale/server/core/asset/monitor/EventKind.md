---
description: Architectural reference for EventKind
---

# EventKind

**Package:** com.hypixel.hytale.server.core.asset.monitor
**Type:** Utility / Type-Safe Enum

## Definition
```java
// Signature
public enum EventKind {
```

## Architecture & Concepts
EventKind is a type-safe enumeration that serves as a critical abstraction layer between the low-level Java NIO file system watching API and the Hytale server's asset monitoring system. Its primary function is to translate the generic, platform-agnostic `java.nio.file.WatchEvent.Kind` objects into a domain-specific, strongly-typed representation.

This decoupling is essential for system stability and maintainability. By abstracting the raw event types, the rest of the asset monitoring pipeline can operate on a clear, self-documenting contract (ENTRY_CREATE, ENTRY_DELETE, ENTRY_MODIFY) without needing knowledge of the underlying `StandardWatchEventKinds` implementation. This prevents implementation details of the Java NIO framework from leaking into higher-level business logic.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (ENTRY_CREATE, ENTRY_DELETE, ENTRY_MODIFY) are constructed by the Java Virtual Machine during class loading. They are compile-time constants and exist before any application code is executed.
- **Scope:** Application-wide. The enum instances persist for the entire lifetime of the server process.
- **Destruction:** The instances are garbage collected only when the class loader is unloaded, which typically occurs during JVM shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. The state of an enum constant is fixed at compile time and cannot be altered at runtime.
- **Thread Safety:** Unconditionally thread-safe. As immutable singletons managed by the JVM, enum instances can be safely accessed and passed between any number of threads without synchronization. The static `parse` method is a pure function with no side effects, also making it completely thread-safe.

## API Surface
The public contract is limited to the enum constants themselves and a single static factory method for translation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(Kind<Path> kind) | static EventKind | O(1) | Translates a Java NIO `Kind` into its corresponding `EventKind`. Throws `IllegalStateException` if the provided kind is unknown or unsupported. |

## Integration Patterns

### Standard Usage
This enum should be used immediately after receiving a file system event from a Java NIO `WatchService` to convert the raw event type into the domain-specific `EventKind`.

```java
// Correct usage within a file watcher service
WatchKey key = watchService.take();
for (WatchEvent<?> event : key.pollEvents()) {
    WatchEvent.Kind<?> rawKind = event.kind();

    // Do not process OVERFLOW events
    if (rawKind == StandardWatchEventKinds.OVERFLOW) {
        continue;
    }

    // Cast and parse into the domain-specific type
    WatchEvent<Path> pathEvent = (WatchEvent<Path>) event;
    EventKind kind = EventKind.parse(pathEvent.kind());

    // Dispatch a typed event to the rest of the system
    assetEventHandler.handle(kind, pathEvent.context());
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Abstraction:** Do not pass raw `StandardWatchEventKinds` objects through the system. This defeats the purpose of this class and tightly couples higher-level logic to the Java NIO API.
- **Uncaught Exceptions:** The `parse` method will throw an `IllegalStateException` for unknown event types. Failure to handle this can crash the asset monitoring thread. Always wrap calls in a try-catch block if forward compatibility with new, unhandled event kinds is a concern.
- **String Comparisons:** Never use the `name()` of the enum for logical comparisons. Rely on direct object equality (`==`) or `switch` statements for performance and type safety.

## Data Pipeline
EventKind acts as a translation and sanitization step early in the asset hot-reloading data pipeline.

> Flow:
> File System I/O Event -> Java NIO WatchService -> `WatchEvent.kind()` -> **EventKind.parse()** -> Domain-Specific Asset Event -> Asset Reloading Logic


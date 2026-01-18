---
description: Architectural reference for PrefabLoadException
---

# PrefabLoadException

**Package:** com.hypixel.hytale.server.core.prefab
**Type:** Transient

## Definition
```java
// Signature
public class PrefabLoadException extends RuntimeException {
```

## Architecture & Concepts
PrefabLoadException is a specialized, unchecked exception that signals a critical failure within the server's prefab loading system. By extending RuntimeException, it indicates a severe, often unrecoverable state, such as a missing file or corrupted data, which typically points to a configuration or deployment error rather than a transient runtime fault.

The core architectural feature is the nested **Type** enum. This provides a structured, machine-readable classification of the failure, distinguishing between a generic processing error (**ERROR**) and a file system lookup failure (**NOT_FOUND**). This design is superior to parsing string-based error messages, allowing for more robust and precise error handling logic in the calling systems.

This exception acts as the primary failure contract for any component that attempts to load, parse, or instantiate a prefab. It is the sole mechanism by which the prefab subsystem communicates fatal errors up the call stack.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the prefab loading machinery when an operation fails. For example, when a requested prefab file does not exist on disk or its contents are malformed.
- **Scope:** Extremely short-lived. An instance of PrefabLoadException exists only for the duration of its propagation up the call stack until it is caught by an error handler or terminates the thread.
- **Destruction:** The object is eligible for garbage collection as soon as it goes out of scope within a `catch` block or after thread termination. It holds no system resources.

## Internal State & Concurrency
- **State:** The internal state, including the failure **Type**, message, and optional cause, is set once during construction and is immutable thereafter. There are no public setters.
- **Thread Safety:** The object is inherently thread-safe due to its immutability. It can be safely passed across threads, although this is not a typical use case for an exception. The act of throwing and catching an exception is a thread-local control flow operation.

## API Surface
The public contract is focused on constructing the exception and inspecting its type.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | PrefabLoadException.Type | O(1) | Returns the enumerated reason for the failure. This is the primary method for inspecting the exception. |

## Integration Patterns

### Standard Usage
Error handling code should catch this specific exception and use the **getType** method to determine the appropriate response, such as logging a detailed error or initiating a graceful server shutdown.

```java
// Correctly handling a prefab loading failure
try {
    Prefab myPrefab = prefabService.load("world_objects/special_tree");
} catch (PrefabLoadException e) {
    // Differentiate behavior based on the failure type
    if (e.getType() == PrefabLoadException.Type.NOT_FOUND) {
        log.error("CRITICAL: Prefab asset is missing. Check server deployment.", e);
        // Potentially trigger a server shutdown
    } else {
        log.error("CRITICAL: Prefab asset is corrupt or unreadable.", e);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing the Exception:** Catching PrefabLoadException and ignoring it will mask critical server configuration errors, leading to unpredictable behavior and cascading failures later.
    ```java
    // BAD: Hides a fatal error
    try {
        loadImportantPrefab();
    } catch (PrefabLoadException e) {
        // This server is now in an inconsistent state
    }
    ```
- **Ignoring the Type:** Catching the exception but failing to inspect its **Type** misses the key value of the class. This leads to generic error messages that are harder to debug.
- **Catching Generic RuntimeException:** Do not write handlers that catch a generic RuntimeException to handle this case. Always catch the most specific exception type possible to avoid accidentally handling unrelated programming errors.

## Data Pipeline
PrefabLoadException does not process data; it represents a terminal point in a data loading pipeline. It is a control flow signal, not a data structure.

> Flow:
> Filesystem I/O Error **or** Data Deserialization Error -> Prefab Loading Service -> `throw new PrefabLoadException(...)` -> Call Stack Propagation -> Exception Handler (e.g., Server Bootstrap Logic) -> Error Logging / System Halt


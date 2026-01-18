---
description: Architectural reference for PrefabSaveException
---

# PrefabSaveException

**Package:** com.hypixel.hytale.server.core.prefab
**Type:** Exception Type

## Definition
```java
// Signature
public class PrefabSaveException extends RuntimeException {
```

## Architecture & Concepts
PrefabSaveException is an *unchecked exception* that signals a critical, non-recoverable failure within the server-side prefab persistence subsystem. By extending RuntimeException, it indicates programming errors or unrecoverable environmental states that a calling method should not be expected to handle gracefully.

Its primary architectural purpose is to provide strongly-typed, structured error information beyond a simple message string. The internal enumeration, Type, allows error handling logic to programmatically differentiate between distinct failure modes, such as a generic I/O error versus a business logic violation like a duplicate name. This avoids fragile string parsing in catch blocks and enables more robust error recovery and reporting.

This exception is a fundamental part of the contract for any API that writes prefab data to a persistent store. It terminates the standard execution flow to prevent the system from proceeding with a corrupted or inconsistent state.

### Lifecycle & Ownership
- **Creation:** Instantiated and thrown exclusively by services responsible for prefab serialization and persistence when a save operation fails. It is created on-demand at the moment of failure.
- **Scope:** Ephemeral. The object's lifetime is bound to the call stack. It exists from the point it is thrown until it is caught by an exception handler or terminates the thread.
- **Destruction:** The object is eligible for garbage collection as soon as the handling `catch` block goes out of scope. No manual resource management is required.

## Internal State & Concurrency
- **State:** The object's state is immutable. Its core data, the failure Type, is set once via the constructor and cannot be changed. All state inherited from RuntimeException (message, cause) is also set at construction.
- **Thread Safety:** As an immutable object, PrefabSaveException is inherently thread-safe. However, it is designed to propagate error information within a single thread of execution. It should not be stored or shared across threads.

## API Surface
The public contract is minimal, focusing on retrieving the structured error type.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | PrefabSaveException.Type | O(1) | Returns the specific category of the save failure, either ERROR or ALREADY_EXISTS. |

## Integration Patterns

### Standard Usage
The intended use is within a `try-catch` block where the handler inspects the exception's type to execute different logic. This allows for specific user feedback or recovery paths.

```java
// How a developer should normally use this
try {
    prefabService.save(myPrefab);
} catch (PrefabSaveException e) {
    if (e.getType() == PrefabSaveException.Type.ALREADY_EXISTS) {
        // Provide specific feedback to the user about a name conflict
        log.warn("Prefab save failed due to a name collision: " + myPrefab.getName());
        userInterface.showError("A prefab with that name already exists.");
    } else {
        // Handle generic, unexpected persistence errors
        log.error("A critical error occurred while saving a prefab.", e);
        // Potentially mark the service as unhealthy
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing the Exception:** Catching this exception and ignoring it is a severe anti-pattern. It masks critical persistence failures and can lead to data loss or a state of desynchronization between the game world and its persistent storage.
- **Catching `RuntimeException`:** Avoid catching the generic parent `RuntimeException` to handle this case. Doing so loses the specific context that PrefabSaveException provides and risks accidentally catching other unrelated runtime errors.
- **Ignoring the Type:** Failing to call `getType()` and treating all instances of this exception identically defeats its primary design purpose. The `Type` enum exists to enable differentiated error handling.

## Data Pipeline
As an exception, this class does not participate in a standard data processing pipeline. Instead, it represents a termination of that pipeline and initiates an error handling flow.

> Flow:
> Prefab Save Request -> Persistence Service -> Validation Failure (e.g., name conflict) or I/O Error -> **PrefabSaveException Instantiation** -> `throw` statement -> Call Stack Unwinding -> `catch` Block -> Error Logging / User Feedback / System Halt


---
description: Architectural reference for PrefabCopyException
---

# PrefabCopyException

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class PrefabCopyException extends Exception {
```

## Architecture & Concepts
PrefabCopyException is a checked exception specifically designed to signal failures within the Prefab manipulation subsystem. Its primary architectural role is to provide a strongly-typed, domain-specific error signal when a prefab copy operation cannot be completed successfully.

This class allows for robust and explicit error handling in the builder tools. Instead of returning nulls or error codes, which can be easily ignored, methods that perform prefab operations are forced to declare that they can throw this exception. This compels the calling code to implement a clear error handling strategy, such as rolling back a transaction or displaying a user-facing error message. It is a critical component for maintaining data integrity and providing a stable user experience in the content creation tools.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand within a method that detects a failure state during a prefab copy process. For example, this could occur if the source prefab data is corrupted or if the target location is invalid.
- **Scope:** The object's lifecycle is extremely brief and confined to the call stack. It exists only from the point it is thrown until it is caught by a corresponding catch block.
- **Destruction:** The object is eligible for garbage collection as soon as the catch block that handles it completes execution. It holds no persistent state and is not managed by any container or registry.

## Internal State & Concurrency
- **State:** Immutable. The exception's state consists of a single error message string, which is set via the constructor at the time of creation and cannot be modified thereafter.
- **Thread Safety:** Inherently thread-safe. As an immutable object with a transient lifecycle, it is typically created, thrown, and caught within the context of a single thread. It does not share state and requires no synchronization mechanisms.

## API Surface
The public contract is minimal, primarily consisting of its constructor and methods inherited from the base java.lang.Exception class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PrefabCopyException(message) | constructor | O(1) | Creates a new instance of the exception with a detailed error message. |

## Integration Patterns

### Standard Usage
The standard pattern is to use a try-catch block to wrap calls to methods that can fail a prefab copy operation. The catch block should handle the failure gracefully.

```java
// How a developer should normally use this
PrefabService service = context.getService(PrefabService.class);
try {
    service.copyPrefab(source, destination);
} catch (PrefabCopyException e) {
    // Log the specific failure and inform the user
    Logger.error("Failed to copy prefab: " + e.getMessage());
    ui.showErrorDialog("Prefab could not be copied.");
}
```

### Anti-Patterns (Do NOT do this)
- **Generic Catching:** Do not catch the generic Exception class. Always catch the specific PrefabCopyException to avoid unintentionally suppressing other, unrelated runtime errors.
- **Ignoring the Exception:** A catch block that is empty or only logs the error without any user feedback is a critical anti-pattern. This hides failures from the user and can lead to a corrupted or inconsistent game state.
- **Control Flow:** Do not use exceptions for non-exceptional control flow. This is a signal for a genuine failure, not a mechanism for branching logic.

## Data Pipeline
This class does not process data in a traditional pipeline. Instead, it acts as a terminal signal that **interrupts** a data flow upon failure.

> Flow:
> User Action (Copy Prefab) -> PrefabService.copyPrefab() -> [Validation Fails] -> **PrefabCopyException Thrown** -> Catch Block -> UI Error Message


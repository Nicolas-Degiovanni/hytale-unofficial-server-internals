---
description: Architectural reference for CircularDependencyException
---

# CircularDependencyException

**Package:** com.hypixel.hytale.assetstore.iterator
**Type:** Transient

## Definition
```java
// Signature
public class CircularDependencyException extends RuntimeException {
```

## Architecture & Concepts
The CircularDependencyException is a critical failure signal within the Hytale Asset Loading subsystem. It is not a general-purpose exception; its sole function is to terminate the asset loading process when a non-recoverable configuration error is detected in the dependency graph of AssetStores.

This exception is thrown by the AssetStoreIterator when it enters a deadlocked state: it has unprocessed AssetStores remaining, but cannot make forward progress because each remaining store is waiting for another store in the same unprocessed set. This indicates a logical contradiction in the asset definitions, such as Asset A requiring Asset B to be loaded first, while Asset B simultaneously requires Asset A.

The primary value of this class is its diagnostic capability. Upon creation, it introspects the state of the AssetStoreIterator and the deadlocked AssetStores to generate a detailed, human-readable error message. This message pinpoints the specific stores involved in the cycle, enabling developers to quickly identify and resolve the configuration error.

## Lifecycle & Ownership
- **Creation:** Instantiated and thrown exclusively by the AssetStoreIterator. This occurs only when the iterator has exhausted all possible loading paths but still has pending stores.
- **Scope:** Ephemeral. The object's lifetime is confined to the stack unwind process, from the point it is thrown until it is caught by a top-level error handler in the engine's bootstrap sequence.
- **Destruction:** The object is eligible for garbage collection immediately after its corresponding catch block completes execution.

## Internal State & Concurrency
- **State:** **Immutable**. The exception's state, which is its detailed error message, is computed once within the constructor via the static makeMessage helper. It holds no mutable fields and its internal data cannot be changed after instantiation.
- **Thread Safety:** Inherently **Thread-Safe**. As an immutable object, it can be safely examined from any thread without synchronization. In practice, it is almost always handled within the single thread responsible for the main asset loading sequence.

## API Surface
The public contract of this class is its constructor. It is not intended to be called directly by client code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CircularDependencyException(values, iterator) | constructor | O(N*M) | Constructs the exception and its detailed diagnostic message. Complexity is proportional to the number of waiting stores (N) and their respective dependencies (M). |

## Integration Patterns

### Standard Usage
This exception is not meant to be thrown by developers. The standard interaction is to catch it at a high level to prevent an ungraceful crash and provide clear feedback.

```java
// Located in a high-level engine initialization service
try {
    assetLoadingService.loadAllStores();
} catch (CircularDependencyException e) {
    // This is a fatal, unrecoverable configuration error.
    // Log the detailed message and terminate the application.
    LOGGER.fatal("Failed to initialize game assets due to a circular dependency.", e);
    Engine.requestShutdown("Asset configuration error.");
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing The Exception:** Never catch this exception and ignore it. It signals a fundamentally broken asset configuration. Continuing execution will result in unpredictable NullPointerExceptions and other critical failures when game systems request assets that could not be loaded.
- **Manual Instantiation:** Do not use `new CircularDependencyException()`. The diagnostic message relies on the precise internal state of the AssetStoreIterator at the moment of deadlock. Creating it manually provides no meaningful information and subverts the asset loading framework.

## Data Pipeline
This class acts as a terminal node in a data processing flow, halting it and reporting a fatal error.

> Flow:
> AssetStoreIterator Deadlock Detection -> **CircularDependencyException Instantiation** -> Stack Unwind -> Engine Exception Handler -> Application Termination


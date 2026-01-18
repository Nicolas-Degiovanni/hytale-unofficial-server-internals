---
description: Architectural reference for ScanResult
---

# ScanResult

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Contract / Value Object

## Definition
```java
// Signature
public interface ScanResult {
```

## Architecture & Concepts
The ScanResult interface is a core component of the world generation and prop placement system. It serves as a contract for representing the outcome of a spatial or logical scanning operation. Its primary architectural purpose is to enforce the **Null Object Pattern**, a design pattern that simplifies client code by providing a non-null, neutral-behavior object in place of a null reference.

In the context of world generation, a "scan" might check if a specific location is suitable for placing a tree, a rock, or a structure. Instead of returning a null value to signify failure (e.g., "no suitable location found"), the scanning operation returns the special `ScanResult.NONE` instance. This design eliminates the need for repetitive null checks throughout the generator's codebase, reducing the risk of NullPointerExceptions and improving code clarity.

Any class that implements ScanResult is expected to be a simple, immutable data-carrier representing the result of a single, atomic operation.

### Lifecycle & Ownership
- **Creation:** Implementations of ScanResult are typically created and returned by methods within the world generation pipeline. The special static instance, `ScanResult.NONE`, is a singleton instantiated by the JVM during class loading.
- **Scope:** The `ScanResult.NONE` instance is application-scoped and persists for the lifetime of the application. All other implementations are expected to be short-lived, transient objects, scoped to the specific generation task that created them.
- **Destruction:** The `ScanResult.NONE` singleton is never garbage collected. Other instances are eligible for garbage collection as soon as they are no longer referenced by the calling code.

## Internal State & Concurrency
- **State:** The interface itself is stateless. The provided `ScanResult.NONE` implementation is a stateless, immutable singleton.
    - **WARNING:** All custom implementations of this interface **must** be immutable. Mutable ScanResult objects can introduce severe, difficult-to-debug race conditions and non-deterministic behavior in the multi-threaded world generation environment.
- **Thread Safety:** The `ScanResult.NONE` object is inherently thread-safe due to its immutability. Any class implementing this interface must be designed to be thread-safe, which is best achieved by making it fully immutable.

## API Surface
The public contract is minimal, focusing on a single query to determine the outcome of the operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isNegative() | boolean | O(1) | Returns true if the scan operation was unsuccessful or yielded no result. This is the primary method for control flow. |
| noScanResult() | static ScanResult | O(1) | A static factory method that provides access to the canonical "no result" singleton instance. |

## Integration Patterns

### Standard Usage
The intended pattern is for a system to perform a scan, receive a ScanResult, and immediately check its state to guide subsequent logic. This avoids null checks and leads to cleaner conditional branches.

```java
// A world generator system checks if a prop can be placed
ScanResult result = propPlacementService.scanLocation(x, y, z);

if (result.isNegative()) {
    // Abort placement or try a different location
    return;
}

// Proceed with placement using data from the result
// (Assuming a positive result implementation carries data)
placeProp(result.getPropData());
```

### Anti-Patterns (Do NOT do this)
- **Returning Null:** A method with a ScanResult return type must never return null. This completely defeats the purpose of the pattern. Always return `ScanResult.noScanResult()` to indicate failure.
- **Instance Checking:** Avoid checking `if (result == ScanResult.NONE)`. While functional, it is less idiomatic than using the `isNegative()` method. The interface method provides a layer of abstraction that makes the code more resilient to future changes in the implementation of negative results.
- **Mutable Implementations:** Creating a ScanResult implementation whose state can change after creation is a severe anti-pattern. It breaks the assumption of immutability and can cause catastrophic failures in concurrent generation tasks.

## Data Pipeline
ScanResult acts as a terminal value object in a control flow, not as a conduit in a data pipeline. It signals the end of a query and dictates the next step.

> Flow:
> World Generator -> PropPlacementService.scanLocation() -> **ScanResult** (created) -> World Generator (checks isNegative()) -> Control Flow Diverges (Place Prop or Abort)


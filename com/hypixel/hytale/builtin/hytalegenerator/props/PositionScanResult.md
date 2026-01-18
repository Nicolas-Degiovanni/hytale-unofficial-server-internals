---
description: Architectural reference for PositionScanResult
---

# PositionScanResult

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Transient

## Definition
```java
// Signature
public class PositionScanResult implements ScanResult {
```

## Architecture & Concepts
PositionScanResult is a specialized data structure that encapsulates the outcome of a spatial query within the world generation system. It serves as a concrete implementation of the ScanResult interface, designed specifically to represent a single 3D integer coordinate (Vector3i).

Its primary role is to act as a message or Data Transfer Object (DTO) between a world scanning algorithm and a consumer, such as a prop placement system. The design elegantly handles both success and failure cases within a single object. A successful scan is represented by a valid internal Vector3i, while a failed scan (a "negative" result) is represented by a null internal position. This pattern avoids returning null objects, which can lead to NullPointerExceptions if not handled carefully by the caller.

The class is fundamentally a value object. Its identity is defined by the position it contains, not by its memory address.

## Lifecycle & Ownership
- **Creation:** Instantiated by world generation scanners or prop placement evaluators upon the completion of a search operation. The creator is the system that performed the scan.
- **Scope:** Extremely short-lived. An instance typically exists only for the duration of a single transaction, passing the result from the producer (scanner) to the consumer (placer).
- **Destruction:** Becomes eligible for garbage collection immediately after its data has been consumed. It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state is **effectively immutable**. The constructor and the getPosition method both defensively clone the Vector3i object. This ensures that once a PositionScanResult is created, its internal coordinate cannot be altered by external code, preventing a significant class of bugs related to shared mutable state.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable nature, instances can be safely shared and read by multiple threads without synchronization or locking mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPosition() | Vector3i | O(1) | Returns a clone of the stored position, or null if the scan was negative. **Warning:** Always check for null or use isNegative before calling. |
| cast(ScanResult) | PositionScanResult | O(1) | Safely casts a generic ScanResult to this type. Throws IllegalArgumentException if the provided result is not a PositionScanResult instance. |
| isNegative() | boolean | O(1) | Returns true if the scan failed to find a position (i.e., the internal position is null). This should be the primary method for checking scan success. |

## Integration Patterns

### Standard Usage
The intended pattern involves receiving a generic ScanResult, casting it, checking its status, and then retrieving the data.

```java
// A prop placer receives a generic result from a scanner
ScanResult result = worldScanner.findPlacementLocation();

// Use the static cast method for type safety
PositionScanResult posResult = PositionScanResult.cast(result);

// Always check if the result is negative before accessing data
if (!posResult.isNegative()) {
    Vector3i placementPos = posResult.getPosition();
    // The returned position is guaranteed to be non-null here
    world.placeProp(prop, placementPos);
} else {
    // Handle the case where no suitable position was found
    log.warn("Failed to find placement for prop.");
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Status Checks:** Directly calling getPosition without first calling isNegative is a common source of NullPointerExceptions. The contract requires a status check.
- **Unsafe Casting:** Avoid casting with `(PositionScanResult)result`. The provided static `cast` method includes runtime checks that prevent ClassCastException and should always be preferred.
- **Assuming Mutability:** Attempting to modify the Vector3i returned by getPosition will have no effect on the original PositionScanResult object, as the method returns a clone. Relying on such a side effect is a design error.

## Data Pipeline
PositionScanResult is a critical link in the procedural generation pipeline, carrying data from spatial analysis to world mutation.

> Flow:
> World Generation Algorithm -> Scanner Logic -> **PositionScanResult** -> Prop Placement System -> World State Mutation


---
description: Architectural reference for RotatedPositionsScanResult
---

# RotatedPositionsScanResult

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.directionality
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class RotatedPositionsScanResult implements ScanResult {
```

## Architecture & Concepts
The RotatedPositionsScanResult class is a specialized data structure that encapsulates the outcome of a world generation scanning operation. It serves as a concrete implementation of the ScanResult interface, specifically designed to hold a collection of positions that include rotational data.

Within the Hytale world generator, "scanners" are components responsible for querying volumes of the world to find suitable locations for placing content, such as structures, props, or vegetation. This class acts as the data transfer object (DTO) returned by such scanners when the content being placed is directional. It is a simple, immutable container whose primary purpose is to transport a list of potential placement locations from the scanning subsystem to a placement or decoration subsystem.

Its existence decouples the logic of *finding* locations from the logic of *acting upon* those locations.

### Lifecycle & Ownership
- **Creation:** Instantiated by a world generator scanner component upon the successful completion of a search. The scanner populates a List of RotatedPosition objects and packages it within this result object.
- **Scope:** Extremely short-lived and transient. Its lifecycle is typically confined to the scope of a single world generation tick or a single prop placement operation. It exists only to be returned from one method and consumed by the caller.
- **Destruction:** The object becomes eligible for garbage collection immediately after the consumer has processed its contents (i.e., retrieved the list of positions). It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The core state is the public final field named positions, which is a List of RotatedPosition objects. The reference to this list is immutable.
- **Thread Safety:** This class is conditionally thread-safe. While the object itself has no internal locks and its direct field is final, its overall thread safety is dependent on the List instance provided during construction.
    - **WARNING:** If a mutable List (e.g., ArrayList) is passed to the constructor and a reference to that list is retained and modified by another thread, data races can occur. For guaranteed thread safety, the constructing code should pass an immutable list, such as one created by `List.of()` or wrapped with `Collections.unmodifiableList`.

## API Surface
The public API is minimal, consisting of a public field and two methods designed for type safety and result validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| positions | List<RotatedPosition> | O(1) | Public field providing direct access to the list of results. |
| cast(ScanResult) | static RotatedPositionsScanResult | O(1) | Safely casts a generic ScanResult to this type. Throws IllegalArgumentException on type mismatch. |
| isNegative() | boolean | O(1) | Implements the ScanResult contract. Returns true if the scan yielded no results (list is null or empty). |

## Integration Patterns

### Standard Usage
The intended pattern is for a consumer to receive a generic ScanResult, validate its type using the static cast method, check if the result is negative, and then process the list of positions.

```java
// A world generator component receives a generic result from a scanner
ScanResult genericResult = worldScanner.findPlacementSpots();

// Safely cast to the expected type
RotatedPositionsScanResult directionalResult = RotatedPositionsScanResult.cast(genericResult);

// Check for a failed or empty scan before processing
if (!directionalResult.isNegative()) {
    for (RotatedPosition pos : directionalResult.positions) {
        // Place a directional block or prop at the given position and rotation
        world.setBlock(pos.x, pos.y, pos.z, Blocks.STAIRS, pos.rotation);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Casting:** Do not use a raw `instanceof` check and manual cast. The static `cast` method is the sanctioned and safer approach, providing a single point of failure with a clear exception.
- **Ignoring isNegative:** Directly accessing the `positions` field without first calling `isNegative` is unsafe. A failed scan could result in a null or empty list, leading to NullPointerExceptions or unnecessary iteration.
- **Result Modification:** The consumer must not attempt to modify the `positions` list. This object represents a final, immutable result. Modifying it violates the data transfer contract and can lead to unpredictable behavior in the world generator.

## Data Pipeline
This class is a critical link in the procedural generation data flow, carrying structured information from a low-level search algorithm to a higher-level placement system.

> Flow:
> World Generation Prop -> Scanner Subsystem -> **RotatedPositionsScanResult** -> Prop Placer Logic -> World State Modification


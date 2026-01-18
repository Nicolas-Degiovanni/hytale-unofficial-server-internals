---
description: Architectural reference for PositionListScanResult
---

# PositionListScanResult

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Transient Data Container

## Definition
```java
// Signature
public class PositionListScanResult implements ScanResult {
```

## Architecture & Concepts
PositionListScanResult is a specialized data transfer object used within the Hytale world generation system. It serves as a concrete implementation of the ScanResult interface, designed to encapsulate the outcome of a spatial query that searches for a set of suitable coordinates.

Its primary role is to act as a message, carrying a list of Vector3i positions from a "scanner" component to a "placer" component. A scanner might search a world chunk for all valid locations to spawn a cluster of trees, and this class would hold those locations. The core design principle is to decouple the logic of *finding* locations from the logic of *acting upon* them.

The class adheres to the ScanResult contract, which provides a unified way to determine if a generation step was successful via the isNegative method. For this class, a "negative" result signifies that no suitable positions were found.

## Lifecycle & Ownership
- **Creation:** Instantiated by a world generation scanner component immediately after it completes a spatial query. The constructor is passed the list of discovered coordinates.
- **Scope:** Ephemeral. The object's lifetime is extremely short, typically confined to a single prop placement operation within the world generator's execution frame.
- **Destruction:** The object has no managed resources and is eligible for garbage collection as soon as the consuming system (e.g., a prop placer) has processed the position list.

## Internal State & Concurrency
- **State:** Mutable. The class holds a direct reference to the List of Vector3i objects passed into its constructor. While the class itself provides no methods to alter the list, the underlying List object can be modified by any component that holds a reference to it. It is architecturally expected that this object is treated as immutable after creation.

- **Thread Safety:** **This class is not thread-safe.** It performs no internal locking. It is designed to be created, passed, and consumed within a single, isolated world generation thread or task. Sharing instances of this class across multiple threads without external synchronization will lead to race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPositions() | List<Vector3i> | O(1) | Returns the list of discovered positions. May return null if the scan was negative from the start. |
| cast(ScanResult) | PositionListScanResult | O(1) | Static utility to safely downcast a generic ScanResult. Throws IllegalArgumentException if the provided object is not an instance of PositionListScanResult. |
| isNegative() | boolean | O(1) | Returns true if the internal position list is null or empty, indicating a failed or fruitless scan. This is the primary method for checking the scan's success. |

## Integration Patterns

### Standard Usage
This class is used as part of a producer-consumer pattern within the world generator. A scanner produces the result, and a placer consumes it, checking for a positive result before acting.

```java
// A scanner component produces the result
List<Vector3i> foundSpots = worldScanner.findLocationsFor(propDefinition);
ScanResult result = new PositionListScanResult(foundSpots);

// A placer component consumes the result
if (!result.isNegative()) {
    // The cast method ensures type safety before accessing specific data
    PositionListScanResult positionResult = PositionListScanResult.cast(result);
    List<Vector3i> positionsToUse = positionResult.getPositions();

    for (Vector3i pos : positionsToUse) {
        world.placeObject(prop, pos);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **External Modification:** Do not modify the list obtained from getPositions. This can cause unpredictable side effects if the list is referenced elsewhere in the generation pipeline. The object should be treated as read-only.
- **Cross-Thread Sharing:** Never pass an instance of PositionListScanResult from one thread to another without a proper memory barrier or synchronization. It is intended for thread-local use.
- **Incorrect Casting:** Do not perform an unsafe cast. Always use the static cast method to ensure type integrity and prevent ClassCastException.

## Data Pipeline
The class is a simple data vessel in a larger generation pipeline. Its flow is linear and unidirectional.

> Flow:
> World Generation Scanner -> Raw List<Vector3i> -> **PositionListScanResult** -> Prop Placement Logic -> World State Modification


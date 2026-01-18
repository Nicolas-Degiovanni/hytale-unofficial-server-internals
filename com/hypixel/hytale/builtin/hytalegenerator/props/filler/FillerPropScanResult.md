---
description: Architectural reference for FillerPropScanResult
---

# FillerPropScanResult

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.filler
**Type:** Transient Data Object

## Definition
```java
// Signature
public class FillerPropScanResult implements ScanResult {
```

## Architecture & Concepts
The FillerPropScanResult is a specialized Data Transfer Object (DTO) used within the procedural world generation engine. It serves as a container for the output of a specific type of environmental scan: one that identifies a collection of block positions suitable for a "filler" operation, such as creating a pool of water or a patch of lava.

This class is a concrete implementation of the generic ScanResult interface. This design allows the world generator's core logic to operate on the abstract ScanResult, while specific prop applicators can safely downcast to this type to access the detailed coordinate data. The primary role of this object is to decouple the *scanning* phase (identifying locations) from the *placement* phase (modifying world data) in the generation pipeline.

The concept of a "negative" result, implemented via the isNegative method, is a critical part of the contract. It provides a standardized, low-cost way for the generation system to determine if a scan was successful without needing to inspect the payload, thereby avoiding null checks and potential exceptions.

## Lifecycle & Ownership
- **Creation:** Instantiated by a world generation scanner component (e.g., a FluidPropScanner) after it analyzes a region of the world. The scanner populates the object with a list of coordinates that meet the placement criteria.
- **Scope:** Extremely short-lived. An instance of FillerPropScanResult typically exists only for the duration of a single prop placement operation. It is created, passed to a prop placement system, and immediately becomes eligible for garbage collection after its data is consumed.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods associated with this object.

## Internal State & Concurrency
- **State:** Effectively immutable after construction. The internal list of positions is assigned once in the constructor. While the List object itself could be mutable if a mutable implementation is passed in, the FillerPropScanResult class does not expose any methods to modify its internal state. It is a simple data carrier.
- **Thread Safety:** **Not thread-safe.** This object is designed to be created and used within the context of a single world generation thread. It contains no internal synchronization mechanisms. Sharing an instance across multiple threads is an anti-pattern and will lead to unpredictable behavior if the underlying list is modified.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFluidBlocks() | List<Vector3i> | O(1) | Returns the list of block positions identified by the scan. May return null if the scan was negative. |
| cast(ScanResult) | FillerPropScanResult | O(1) | Static utility to safely downcast a generic ScanResult. Throws IllegalArgumentException if the provided result is not an instance of FillerPropScanResult. |
| isNegative() | boolean | O(1) | Returns true if the scan failed to find any valid positions (i.e., the internal list is null or empty). This check should always be performed before attempting to access the data. |

## Integration Patterns

### Standard Usage
The intended pattern involves a scanner producing the result and a consumer (placer) checking its validity before using the data. The static cast method ensures type safety.

```java
// In a prop placement system
ScanResult genericResult = worldScanner.findLocationsForFluid();

if (!genericResult.isNegative()) {
    // Use the safe cast method, not a direct Java cast
    FillerPropScanResult fluidResult = FillerPropScanResult.cast(genericResult);
    List<Vector3i> positionsToFill = fluidResult.getFluidBlocks();

    // The list is guaranteed not to be null or empty here
    for (Vector3i pos : positionsToFill) {
        world.setBlock(pos, Blocks.WATER);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring isNegative:** Directly calling getFluidBlocks without first checking isNegative can result in a NullPointerException if the scan was unsuccessful.

- **Unsafe Casting:** Do not use a direct Java cast. The provided static cast method is the contract for ensuring type compatibility within the prop system.
    ```java
    // BAD: Unsafe, can throw ClassCastException
    FillerPropScanResult result = (FillerPropScanResult) genericResult;
    ```

- **External State Mutation:** The list returned by getFluidBlocks should be treated as read-only. Modifying it can lead to unpredictable side effects if the result object is referenced elsewhere in the generation pipeline.

## Data Pipeline
This object acts as a message carrying data from one stage of world generation to the next.

> Flow:
> World Chunk Data -> Prop Scanner -> **FillerPropScanResult** (Payload: List<Vector3i>) -> Prop Placement System -> World Block Update


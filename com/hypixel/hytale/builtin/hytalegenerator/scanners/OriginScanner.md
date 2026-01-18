---
description: Architectural reference for OriginScanner
---

# OriginScanner

**Package:** com.hypixel.hytale.builtin.hytalegenerator.scanners
**Type:** Singleton

## Definition
```java
// Signature
public class OriginScanner extends Scanner {
```

## Architecture & Concepts
The OriginScanner is a foundational and highly specialized implementation of the Scanner strategy. Its role within the world generation framework is to perform a single point-check, not a volumetric scan. It answers the simple, boolean question: "Does the provided Pattern match at this exact coordinate?"

Unlike other scanners that might iterate over a volume to find all valid placement locations, the OriginScanner is an atomic predicate. It operates on the single `position` provided in the `Scanner.Context` and immediately returns. This makes it extremely fast and predictable.

It serves as a critical building block for procedural generation tasks where a feature's placement is predetermined or needs to be validated at a specific origin point before placement. It is the most granular scanner available, representing the base case for more complex scanning and placement logic.

## Lifecycle & Ownership
- **Creation:** The OriginScanner is an eagerly instantiated singleton. A single `instance` is created by the JVM class loader when the class is first referenced and is stored in a private static final field.
- **Scope:** Application-scoped. The single instance persists for the entire lifetime of the server or client process. It is shared globally.
- **Destruction:** The object is not explicitly destroyed. It is garbage collected by the JVM during the final process shutdown.

## Internal State & Concurrency
- **State:** The OriginScanner is stateless and immutable. Its behavior is purely a function of its inputs, specifically the `Scanner.Context` passed to the `scan` method. It holds no internal state that changes between calls.
- **Thread Safety:** This class is inherently thread-safe. Due to its stateless nature, a single instance can be safely and concurrently invoked by any number of world generation worker threads without locks, synchronization, or risk of race conditions. This design is crucial for the performance of a parallelized world generation engine.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(Scanner.Context context) | List<Vector3i> | O(P) | Executes a point-check. Complexity is determined by the `pattern.matches()` call, denoted as P. Returns a singleton list containing the context position on a successful match, or an empty list on failure. |
| scanSpace() | SpaceSize | O(1) | Returns the fixed 1x1x1 bounding box this scanner considers. This value is constant and represents a single block volume. |
| getInstance() | OriginScanner | O(1) | Retrieves the globally shared singleton instance of the scanner. |

## Integration Patterns

### Standard Usage
The OriginScanner should always be retrieved via its static `getInstance` method. It is typically used by higher-level generation logic that needs to validate a specific point rather than search an area.

```java
// In a world generator, after calculating a specific target position
Scanner.Context scannerContext = new Scanner.Context(targetPosition, somePattern, materialSpace, workerId);

Scanner pointChecker = OriginScanner.getInstance();
List<Vector3i> matches = pointChecker.scan(scannerContext);

if (!matches.isEmpty()) {
    // The pattern is valid at targetPosition. Proceed with feature placement.
    world.placeFeature(matches.get(0), someFeature);
}
```

### Anti-Patterns (Do NOT do this)
- **Misuse for Area Scanning:** Do not use the OriginScanner inside a loop to simulate a volumetric scan. This is extremely inefficient and defeats the purpose of more advanced scanners like the GridScanner or VolumeScanner, which are optimized for that task.

- **Ignoring the Result:** The return value of `scan` is the sole indicator of success. Assuming a match without checking if the returned list is empty will lead to invalid feature placement.

## Data Pipeline
The data flow for the OriginScanner is a direct, non-iterative path. It acts as a simple filter or gate in a larger generation process.

> Flow:
> World Generation Algorithm -> `Scanner.Context` (Target Position, Pattern) -> **OriginScanner.scan()** -> `Pattern.matches()` -> List<Vector3i> (Size 0 or 1) -> World Generation Algorithm


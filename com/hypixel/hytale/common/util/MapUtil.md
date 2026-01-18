---
description: Architectural reference for MapUtil
---

# MapUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class MapUtil {
```

## Architecture & Concepts
MapUtil is a stateless, static utility class designed to provide high-performance, consistent, and safe map combination operations. It serves as a centralized component for merging map data structures throughout the Hytale engine.

The primary architectural driver for this class is the enforcement of best practices for data handling. By providing methods that return unmodifiable maps, it encourages defensive programming and the use of immutable data structures. This is critical in a complex, multi-threaded environment like a game engine to prevent state corruption and race conditions.

The default use of `it.unimi.dsi.fastutil.objects.Object2ObjectOpenHashMap` is a deliberate performance choice. This signals a system-wide preference for memory-efficient and fast collections over standard Java Development Kit collections for performance-critical paths. The class also provides overrides that accept a `Supplier`, allowing developers to use other map implementations (e.g., `LinkedHashMap`) when specific behaviors like insertion-order preservation are required.

## Lifecycle & Ownership
- **Creation:** Not applicable. MapUtil is a stateless utility class composed exclusively of static methods and cannot be instantiated.
- **Scope:** The methods of this class are available for the entire application runtime, bound to the lifecycle of the ClassLoader that loads it.
- **Destruction:** Not applicable. As no instances are created, no resource cleanup or destruction is necessary.

## Internal State & Concurrency
- **State:** **Stateless**. This class holds no internal state, either static or instance-based. All operations are pure functions that operate exclusively on their input arguments, producing a new map instance as output.
- **Thread Safety:** **Inherently Thread-Safe**. As a stateless utility, its methods can be invoked concurrently from any thread without risk of data corruption. The thread safety of the *returned* map is determined by the specific method called. Methods returning `Collections.unmodifiableMap` provide a read-only, thread-safe view of the combined data.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| combineUnmodifiable(one, two) | Map<T, V> | O(n + m) | Merges two maps into a new, high-performance map and returns an immutable view. Keys from the `two` map will overwrite keys from the `one` map. |
| combineUnmodifiable(one, two, supplier) | Map<T, V> | O(n + m) | Merges two maps using a caller-provided map implementation and returns an immutable view. Essential for cases requiring specific map semantics. |
| combine(one, two, supplier) | M extends Map<T, V> | O(n + m) | Merges two maps using a caller-provided map implementation and returns the new **mutable** map. Use this only when subsequent modifications are required. |

## Integration Patterns

### Standard Usage
The primary use case is to safely merge configuration maps, resource tables, or any other key-value data where one set of values should supplement or override another.

```java
// Example: Merging default and user-specific keybindings
Map<String, Action> defaultBinds = ...;
Map<String, Action> userBinds = ...;

// The final map is immutable, preventing accidental changes during gameplay.
// User-defined bindings will correctly overwrite the defaults.
Map<String, Action> finalKeyBinds = MapUtil.combineUnmodifiable(defaultBinds, userBinds);
```

### Anti-Patterns (Do NOT do this)
- **Manual Merging:** Do not implement custom map-merging loops in application code. This leads to boilerplate, potential bugs, and misses the performance benefits of using `fastutil`. Always centralize this logic through MapUtil.
- **Inappropriate Mutability:** Do not use the `combine` method when a read-only result is sufficient. Prefer `combineUnmodifiable` to create immutable data structures, which protects system integrity by preventing unintended downstream mutations.

## Data Pipeline
MapUtil acts as a simple, stateless transformation stage in a data pipeline. It takes two distinct map data sources and transforms them into a single, unified map.

> Flow:
> Map A (e.g., Default Config) -> **MapUtil.combineUnmodifiable** -> Combined Immutable Map
> Map B (e.g., User Overrides) -> **MapUtil.combineUnmodifiable** -> Combined Immutable Map


---
description: Architectural reference for AssetConstants
---

# AssetConstants

**Package:** com.hypixel.hytale.assetstore
**Type:** Utility

## Definition
```java
// Signature
public class AssetConstants {
```

## Architecture & Concepts
The AssetConstants class serves as a centralized, static repository for performance-tuning values within the Hytale asset management system. It does not contain any logic. Instead, it provides compile-time constants used to pre-size and optimize data structures, primarily collections like HashMaps and ArrayLists, throughout the asset loading and storage pipeline.

The primary architectural role of this class is to decouple performance heuristics from the core asset management logic. By externalizing these "magic numbers", we allow engineers to tune memory allocation and computational complexity for asset processing without modifying the algorithms themselves. These constants represent educated guesses about the topology of the asset graph, such as the average number of child assets or tags per asset. Incorrectly tuning these values can lead to memory waste or performance degradation due to excessive collection resizing and hash collisions.

## Lifecycle & Ownership
This class is a static utility and is never instantiated. Its lifecycle is managed directly by the Java Virtual Machine's class loader.

- **Creation:** The class is loaded into memory by the JVM when it is first referenced by another component, typically during the initialization of the AssetStore service. Its static final fields are initialized at this time.
- **Scope:** Application-level. The constants are available globally for the entire duration of the application's runtime.
- **Destruction:** The class is unloaded from memory when the JVM shuts down. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Entirely immutable. All members are public static final primitives, whose values are fixed at compile time.
- **Thread Safety:** Inherently thread-safe. As these are compile-time constants, they can be read from any thread at any time without synchronization. There is no possibility of a race condition or data corruption.

## API Surface
The public contract consists solely of static constants used for performance tuning.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EXPECTED_CHILDREN_PER_ASSET | int | Constant | The default initial capacity for collections storing child assets. |
| EXPECTED_ASSETS_PER_PATH | int | Constant | The default initial capacity for collections mapping a path to its assets. |
| EXPECTED_VALUES_PER_TAG | int | Constant | The default initial capacity for collections storing values associated with a single tag. |
| EXPECTED_ASSETS_PER_TAG | int | Constant | The default initial capacity for collections mapping a tag to its associated assets. |

## Integration Patterns

### Standard Usage
These constants should be used directly when initializing collections within the asset system to prevent costly resizing operations.

```java
// Correctly pre-sizing a map using a constant
import com.hypixel.hytale.assetstore.AssetConstants;

// ... inside a class like AssetTagRegistry
Map<Tag, List<Asset>> tagToAssets = new HashMap<>(
    AssetConstants.EXPECTED_ASSETS_PER_TAG
);
```

### Anti-Patterns (Do NOT do this)
- **Instantiation:** Never create an instance of this class. It provides no value and pollutes the heap. The constructor should be declared private to prevent this.
- **Reflection:** Do not attempt to modify these constants at runtime using reflection. The entire asset system relies on these values for its performance and memory profile. Unpredictable behavior, including severe performance degradation, will occur.
- **Hard-Coding Values:** Avoid using the numeric literals directly in the code. Always reference the constant from this class to maintain a single source of truth for performance tuning.

## Data Pipeline
This class is not an active component in any data pipeline. It does not process, transform, or route data. It serves as a static configuration provider, influencing the performance characteristics of components that *are* in the pipeline, such as the AssetLoader and AssetStore.


---
description: Architectural reference for AssetStoreUtil
---

# AssetStoreUtil

**Package:** com.hypixel.hytale.builtin.asseteditor.util
**Type:** Utility

## Definition
```java
// Signature
public class AssetStoreUtil {
```

## Architecture & Concepts
AssetStoreUtil is a stateless utility class designed to provide reverse lookup capabilities for the Hytale Asset Store system. Its primary function is to translate a low-level integer *asset index* back into its high-level, human-readable string *ID*.

This class acts as a compatibility layer, bridging the gap between performance-oriented, index-based data structures and the canonical, string-based identification system for assets. The implementation relies on runtime type-checking to probe the underlying AssetMap within an AssetStore, determining if it supports indexed lookups. This pattern suggests that the core AssetMap interface does not guarantee an index-to-ID mapping, necessitating this specialized, type-aware utility.

**WARNING:** The entire class is deprecated. Its existence points to a legacy pattern that should be avoided in new systems. Modern asset access should rely on more direct and type-safe mechanisms provided by the AssetStore API, rather than manipulating raw integer indices.

### Lifecycle & Ownership
- **Creation:** This is a stateless utility class and is never instantiated. All methods are static.
- **Scope:** Not applicable. As a collection of static methods, it has no instance lifecycle.
- **Destruction:** Not applicable.

## Internal State & Concurrency
- **State:** Stateless. This class contains no member variables and does not manage any internal state. Its operations are pure functions of their inputs.
- **Thread Safety:** Fully thread-safe. The methods have no side effects and operate exclusively on the provided arguments. Concurrency concerns related to the AssetStore object itself are the responsibility of the caller.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIdFromIndex(assetStore, assetIndex) | String | O(1) | **DEPRECATED.** Retrieves the string ID for an asset given its integer index. Throws IllegalArgumentException if the underlying AssetMap does not support indexed lookups. |

## Integration Patterns

### Standard Usage
The following example demonstrates the intended, albeit deprecated, use of this utility. It is provided for maintenance and debugging of existing systems only.

**WARNING:** Do not use this utility in new code. A direct, non-indexed lookup mechanism should be preferred.

```java
// This example is for historical and debugging purposes only.
// The use of AssetStoreUtil is strongly discouraged in new code.

AssetStore blockStore = context.getAssetStore(BLOCKS);
int blockIndex = 15; // An index obtained from world data

try {
    // Perform a reverse lookup to get the asset's string identifier
    String blockId = AssetStoreUtil.getIdFromIndex(blockStore, blockIndex);
    System.out.println("Block at index 15 has ID: " + blockId);
} catch (IllegalArgumentException e) {
    // This occurs if the asset store is not one of the recognized indexed types
    System.err.println("Error: The provided asset store does not support index lookups.");
}
```

### Anti-Patterns (Do NOT do this)
- **Usage in New Systems:** The primary anti-pattern is using this class at all. Its deprecated status signals that its functionality is obsolete and has been replaced by more robust patterns within the engine's asset management system.
- **Ignoring Exceptions:** Callers must not assume an AssetStore supports indexed lookups. Failing to catch the potential IllegalArgumentException will lead to runtime crashes when encountering non-indexed asset types, such as those using hash maps for storage.
- **Performance-Critical Loops:** While the operation is O(1), the multiple `instanceof` checks introduce minor overhead. In a tight loop, this is less efficient than using a system designed from the ground up for index-based access.

## Data Pipeline
This utility functions as a simple transformer in a data pipeline, converting a numeric identifier into a string identifier by querying a data store.

> Flow:
> Integer Index -> **AssetStoreUtil.getIdFromIndex** -> AssetStore -> Indexed AssetMap Lookup -> Asset ID String


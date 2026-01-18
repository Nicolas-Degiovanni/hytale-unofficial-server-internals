---
description: Architectural reference for TranslationMap
---

# TranslationMap

**Package:** com.hypixel.hytale.server.core.modules.i18n.generator
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class TranslationMap {
```

## Architecture & Concepts
The TranslationMap class is a specialized, stateful container designed to represent and manipulate internationalization (i18n) key-value pairs. It serves as a foundational component within the server's i18n generation pipeline, which is responsible for processing and organizing language files.

Architecturally, this class is an abstraction over a standard Java LinkedHashMap, chosen specifically to preserve the insertion order of translation keys. This is critical for maintaining human-readable and diff-friendly language files. Its primary role is not for high-performance runtime lookups within the game client, but rather for build-time or server-side tooling that loads, merges, sorts, and prepares translation data before it is consumed by the engine.

The inclusion of specialized manipulation methods, such as putAbsentKeys and sortByKeyBeforeFirstDot, clearly defines its purpose as a data transformation tool, not merely a passive data holder. The custom sorting logic, which groups keys by their prefix before the first dot, is a key feature for enforcing a consistent and logical structure across all generated language files.

## Lifecycle & Ownership
- **Creation:** A TranslationMap is instantiated on-demand by higher-level services in the i18n generation system. Common creation triggers include reading a language properties file from disk or initializing a new, empty set of translations to be populated programmatically. The constructors facilitate loading from existing Map or Properties objects.

- **Scope:** The lifetime of a TranslationMap instance is ephemeral and strictly bound to the scope of the operation that created it. For example, an instance might exist only for the duration of a single file-processing task. It does not persist across sessions or server restarts.

- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the generation task completes and all references to the instance are released. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. The core data structure is a LinkedHashMap that is directly modified by most methods, including put, removeKeys, and putAbsentKeys. The sortByKeyBeforeFirstDot method is particularly impactful, as it completely reconstructs and replaces the internal map to re-order its entries.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms such as locks or concurrent collections.

    **WARNING:** Concurrent modification of a TranslationMap instance from multiple threads will result in a corrupted state, data loss, and a high probability of a ConcurrentModificationException. Any multi-threaded access must be protected by external synchronization, or preferably, the object's use should be confined to a single thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(key) | String | O(1) | Retrieves the value for a given key. |
| put(key, value) | void | O(1) | Adds or overwrites a key-value pair. |
| removeKeys(keys) | void | O(N) | Performs a bulk removal of keys. N is the size of the input collection. |
| putAbsentKeys(other) | void | O(M) | Merges another map, only adding keys that do not already exist. M is the size of the other map. |
| sortByKeyBeforeFirstDot() | void | O(N log N) | Re-orders the entire map in-place based on a hierarchical key-sorting algorithm. This is a computationally expensive operation. |
| asMap() | Map | O(1) | Returns a read-only, unmodifiable view of the internal map. |

## Integration Patterns

### Standard Usage
TranslationMap is designed to be used in a sequential pipeline: load data, transform it, and then extract the result. The asMap method should be used to safely pass the final, processed data to other systems, such as a file writer.

```java
// Example: Merging a default and a custom language map
TranslationMap defaultLang = new TranslationMap(loadProperties("en_us.properties"));
TranslationMap customLang = new TranslationMap(loadProperties("custom.properties"));

// Add any missing keys from the default map into the custom one
customLang.putAbsentKeys(defaultLang);

// Enforce a standard key ordering for consistency
customLang.sortByKeyBeforeFirstDot();

// The result can now be written to a file or processed further
Map<String, String> finalTranslations = customLang.asMap();
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Sharing:** Never share a single TranslationMap instance across multiple threads without external locking. Its mutable, unsynchronized nature makes it inherently unsafe for concurrent use.

- **Runtime Lookups:** Do not use this class for frequent translation lookups in a performance-critical context, such as the game client's rendering loop. Its mutable design and expensive sorting methods are ill-suited for high-throughput, read-only operations. A dedicated, immutable data structure is used for that purpose.

## Data Pipeline
The TranslationMap acts as a central processing stage within the i18n asset generation pipeline. Data flows into it for manipulation and then flows out to be serialized.

> Flow:
> Raw Language File (.properties) -> File Parser -> **TranslationMap** (Load/Merge/Sort) -> Unmodifiable Map -> Serializer -> Final Game Asset


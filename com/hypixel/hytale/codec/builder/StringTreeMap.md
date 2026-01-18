---
description: Architectural reference for StringTreeMap
---

# StringTreeMap

**Package:** com.hypixel.hytale.codec.builder
**Type:** Utility / Data Structure

## Definition
```java
// Signature
public class StringTreeMap<V> {
```

## Architecture & Concepts
The StringTreeMap is a specialized, high-performance implementation of a Radix Tree, also known as a Trie. Its primary function is to accelerate the decoding of structured data, particularly JSON, by providing extremely fast key lookups directly against a raw data stream.

Unlike a standard HashMap which would require allocating a String for a key before performing a lookup, the StringTreeMap is designed to integrate directly with the **RawJsonReader**. This allows the system to traverse the tree structure character-by-character from the input stream, avoiding intermediate String allocations for keys entirely. This pattern is critical for performance in memory-sensitive, high-throughput environments like network packet decoding.

The core optimization is the conversion of four-character segments of a key into a single primitive **long**. This **long** value is then used as the key in an underlying `fastutil` Long2ObjectMap, which is significantly more performant and memory-efficient than using String objects as keys. Each node in the tree represents a four-character prefix, allowing for rapid traversal down the tree as the key is read from the source.

This class is a foundational component of the Hytale codec system, acting as a pre-compiled lookup table for known object fields during deserialization.

### Lifecycle & Ownership
-   **Creation:** A StringTreeMap is instantiated on-demand. It is typically constructed once with a known set of keys (e.g., the field names of a serializable class) and then reused for many decoding operations. The constructor accepting a `Map<String, V>` is the primary entry point for this pattern.
-   **Scope:** The lifetime of a StringTreeMap instance is entirely managed by its owner. In a typical codec scenario, it would be held as a static final field within a specific deserializer, making its scope application-wide and permanent after class initialization.
-   **Destruction:** The object is subject to standard Java garbage collection. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The StringTreeMap is a mutable, stateful data structure. Its internal tree can be modified via the `put` and `remove` methods. The structure is built once and is typically treated as read-only thereafter.
-   **Thread Safety:** **This class is not thread-safe.** All internal data structures, including the `Long2ObjectOpenHashMap`, are accessed without any synchronization. Concurrent writes will corrupt the tree, and concurrent reads and writes will lead to undefined behavior. If concurrent access is required, it must be managed externally with explicit locking.

## API Surface
The public API is minimal, focusing on building the tree and performing lookups from a **RawJsonReader**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| put(key, value) | void | O(L) | Inserts a key-value pair. Complexity is proportional to the key length L. |
| remove(key) | void | O(L) | Removes a key-value pair. Complexity is proportional to the key length L. |
| findEntry(reader) | StringTreeMap | O(L) | The primary lookup method. Traverses the tree using the input reader. Returns the terminal node or null. Throws IOException. |
| getValue() | V | O(1) | Returns the value stored at this specific node. |
| getKey() | String | O(1) | Returns the full key stored at this specific node. |

## Integration Patterns

### Standard Usage
The intended use is to pre-build a map for a specific data structure and reuse it within a deserializer to quickly map incoming JSON keys to corresponding field setters or handlers.

```java
// 1. Pre-build the map from known field names
Map<String, FieldSetter> fields = new HashMap<>();
fields.put("playerName", player::setName);
fields.put("playerScore", player::setScore);
StringTreeMap<FieldSetter> fieldMap = new StringTreeMap<>(fields);

// 2. During deserialization, use it with a RawJsonReader
// RawJsonReader reader = ... points to a JSON key like "playerName"
StringTreeMap<FieldSetter> node = fieldMap.findEntry(reader);

if (node != null) {
    FieldSetter setter = node.getValue();
    // read value from reader and apply
    setter.apply(readValue(reader));
}
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Do not modify the tree from one thread while another thread is calling `findEntry`. This will cause catastrophic failures. The tree should be built once and then treated as immutable.
-   **General Purpose Map:** Do not use StringTreeMap as a generic replacement for a `java.util.HashMap`. Its performance benefits are tied directly to its integration with **RawJsonReader**. For general-purpose key-value storage, a standard HashMap is simpler and more appropriate.
-   **Dynamic Key Sets:** This structure is not optimized for scenarios where keys are frequently added and removed. It is designed for a static, pre-defined set of keys.

## Data Pipeline
The StringTreeMap is not a data processor itself, but a critical lookup component within a larger decoding pipeline.

> Flow:
> Raw JSON Stream -> RawJsonReader -> **StringTreeMap.findEntry()** -> Value Deserializer -> Target Object Field


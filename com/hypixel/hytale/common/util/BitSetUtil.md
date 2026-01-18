---
description: Architectural reference for BitSetUtil
---

# BitSetUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class BitSetUtil {
```

## Architecture & Concepts
BitSetUtil is a high-performance, low-level utility class designed for a single purpose: to perform an extremely fast copy of the contents of one java.util.BitSet to another.

Architecturally, this class represents a deliberate decision to sacrifice portability and safety for raw performance. It operates by completely bypassing the public API of the standard BitSet class. Instead of iterating over bits, it uses sun.misc.Unsafe to directly access the private internal fields of the BitSet object—specifically the `words` array (a `long[]`) and the `wordsInUse` integer. By treating the BitSet as a simple memory structure, it can perform a bulk memory copy of its underlying data.

This component is a critical optimization tool, intended for use only in performance-sensitive systems where BitSet copy operations are a known bottleneck. Common use cases include chunk data serialization, entity component state replication, or any system that frequently duplicates large bitmasks.

**WARNING:** This class is inherently fragile. Its functionality is tightly coupled to the internal implementation of the `java.util.BitSet` class in the specific version of the JDK being used. Any changes to the private field names or types in a future Java update will cause this utility to fail at runtime.

### Lifecycle & Ownership
- **Creation:** As a static utility class, BitSetUtil is never instantiated. Its static fields, which hold the memory offsets of the BitSet internal fields, are initialized once by the JVM ClassLoader when the class is first loaded.
- **Scope:** The class and its static state are application-scoped, persisting for the entire lifetime of the JVM process.
- **Destruction:** The class is unloaded when its ClassLoader is garbage collected, which typically occurs only at JVM shutdown.

## Internal State & Concurrency
- **State:** The class itself is stateless from a caller's perspective. Its only internal state consists of two `static final` fields, WORDS_OFFSET and WORDS_IN_USE_OFFSET. This state is immutable after the static initializer block completes execution.
- **Thread Safety:** The methods are not internally synchronized. They are thread-safe only to the extent that the caller guarantees that no other thread is concurrently modifying the `from` or `to` BitSet instances. The responsibility for managing concurrent access to the BitSet objects rests entirely with the caller. The underlying memory copy operations are atomic, but this does not prevent higher-level data races.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| copyValues(BitSet from, BitSet to) | void | O(N) | Performs a direct memory copy of the internal state from the `from` BitSet to the `to` BitSet. N is the number of `long`s in use, not the total number of bits. This is significantly faster than the standard API. Falls back to a slower, safer method if Unsafe is unavailable. |
| copyValuesSlow(BitSet from, BitSet to) | void | O(M) | A fallback implementation using the public BitSet API (`clear` and `or`). M is the logical length of the BitSet. This method is guaranteed to be safe and portable but is less performant. |

## Integration Patterns

### Standard Usage
This utility should be invoked directly where a high-performance copy of a BitSet is required. It is a drop-in replacement for the common `destination.clear(); destination.or(source);` pattern.

```java
// How a developer should normally use this
BitSet sourceBitSet = getSourceData();
BitSet destinationBitSet = new BitSet();

// Replaces a multi-operation, slower copy with a single, high-speed one.
BitSetUtil.copyValues(sourceBitSet, destinationBitSet);
```

### Anti-Patterns (Do NOT do this)
- **Cross-JVM Reliance:** Do not write code that assumes this utility will function on a different JVM or a future version of Java without extensive testing. The use of Unsafe makes it highly implementation-dependent.
- **Ignoring the Fallback:** Do not assume the high-performance path is always taken. In environments with a SecurityManager or where Unsafe is disabled, the system will gracefully degrade to the slower, safer implementation. Code must remain correct in both scenarios.
- **Concurrent Modification:** Never call `copyValues` on a BitSet that is being written to by another thread without external synchronization. The result is undefined and will likely lead to a corrupted state.

## Data Pipeline
This utility does not participate in a complex data pipeline. It is a direct, low-level state transfer function. The flow bypasses all logical abstractions of the BitSet class.

> Flow:
> BitSet `from` (Internal `long[]` words) -> Unsafe Memory Read -> System.arraycopy -> Unsafe Memory Write -> BitSet `to` (Internal `long[]` words)


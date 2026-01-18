---
description: Architectural reference for Compressor
---

# Compressor

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.compression
**Type:** Utility

## Definition
```java
// Signature
public class Compressor {
```

## Architecture & Concepts
The Compressor is a stateless, high-performance utility for applying a specialized form of Run-Length Encoding (RLE). Its primary function is to reduce the in-memory footprint of large arrays containing contiguous, repeating object references. This is common in systems like world generation, where large volumes of a single block type may be represented by the same object instance.

The core architectural decision is the use of reference equality (**==**) instead of value equality (*.equals()*). This makes the compression and decompression operations extremely fast, as they avoid potentially expensive equality checks. However, it also means the compressor is only effective on data structures where repeated elements are guaranteed to be the *exact same object instance*.

A key optimization is the **MIN_RUN** threshold, hardcoded to 7. The compressor will not encode runs of identical objects shorter than this length. This avoids the overhead of creating a Run object for sequences so short that the compressed representation would be larger or offer negligible savings compared to the original.

The output is not a raw byte array but a structured Compressor.CompressedArray object, which encapsulates the compressed data and the original, uncompressed length. This metadata is essential for correctly pre-allocating memory during decompression.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its default constructor, `new Compressor()`. It has no dependencies and requires no special context for creation.
- **Scope:** Intended to be a short-lived, transient object. A single instance can be created, used for one or more compression/decompression operations, and then discarded by the garbage collector.
- **Destruction:** The object is eligible for garbage collection as soon as it falls out of scope. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The Compressor is entirely stateless. The MIN_RUN field is a final constant. All operations are pure functions that depend only on their input arguments, producing a new output without any side effects or modification of instance state.
- **Thread Safety:** This class is inherently thread-safe. Due to its stateless nature, a single shared instance of Compressor can be safely used by multiple threads concurrently without locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compressOnReference(T[] in) | Compressor.CompressedArray | O(N) | Compresses the input array. N is the length of the input. Throws NullPointerException if the input array is null. |
| decompress(CompressedArray<T> in) | T[] | O(M) | Decompresses the data into a new array. M is the length of the *original*, uncompressed data. Throws NullPointerException if the input is null. |

## Integration Patterns

### Standard Usage
The Compressor should be instantiated on-demand to perform a specific data reduction task, typically before serialization or network transfer.

```java
// Example: Compressing world chunk data
BlockType[] rawChunkData = getRawData();
Compressor compressor = new Compressor();

// Compress the data
Compressor.CompressedArray<BlockType> compressedData = compressor.compressOnReference(rawChunkData);

// ... serialize or transmit compressedData ...

// Decompress later
BlockType[] restoredChunkData = compressor.decompress(compressedData);
```

### Anti-Patterns (Do NOT do this)
- **Misuse with Value-Types:** Do not use this compressor on arrays where duplicate elements are distinct object instances, even if they are logically equal via *.equals()*. The reference-based comparison will fail to identify them as a run, resulting in zero compression. This will lead to silent performance degradation and memory waste.
- **Compressing Noisy Data:** Attempting to compress data with few or no repeating object references is inefficient. The algorithm will still traverse the entire array, consuming CPU cycles for no benefit. The MIN_RUN threshold mitigates the worst of this, but the overhead of the initial traversal remains.
- **Retaining Instances:** There is no benefit to retaining a Compressor instance for the entire application lifecycle. Its creation is cheap, and holding onto it can mislead developers into thinking it maintains state. Create it, use it, and let it be garbage collected.

## Data Pipeline
The component serves as a reversible transformation step in a larger data processing pipeline.

**Compression Flow:**
> `T[]` (Raw Array) -> **compressOnReference** -> `Compressor.CompressedArray<T>` (Compressed Representation)

**Decompression Flow:**
> `Compressor.CompressedArray<T>` (Compressed Representation) -> **decompress** -> `T[]` (Identical to Original Raw Array)


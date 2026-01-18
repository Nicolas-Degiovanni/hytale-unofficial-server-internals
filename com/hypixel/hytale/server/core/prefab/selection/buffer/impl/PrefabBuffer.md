---
description: Architectural reference for PrefabBuffer
---

# PrefabBuffer

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer.impl
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class PrefabBuffer {
```

## Architecture & Concepts
The **PrefabBuffer** is a highly optimized, immutable, and serialized in-memory representation of a game world prefab. Its primary design goal is to minimize memory footprint and provide extremely fast, forward-only iteration for placing structures into the world. It acts as a compiled, binary artifact derived from a higher-level prefab definition.

The core architecture is split into two distinct phases: construction and consumption.

1.  **Construction (via PrefabBuffer.Builder):** A prefab's block, entity, and metadata are fed into the **Builder**. The builder serializes this data into a custom, column-major binary format within a Netty **ByteBuf**. This format is heavily optimized using bitmasks (defined in **BlockMaskConstants**) to encode block properties like data type sizes, optional fields (e.g., chance), and rotation into a single short. This eliminates padding and storage for default values, resulting in a dense data structure.

2.  **Consumption (via PrefabBufferAccessor):** Once built, the **PrefabBuffer** is effectively immutable. To read the data, a consumer must create a **PrefabBufferAccessor**. This accessor provides a read-only, iterator-style API (**forEach**) over the underlying binary data, abstracting away the complexity of decoding the bitmasks and variable-length fields. This separation ensures that the original buffer remains untouched and can be safely shared for concurrent reads.

This component is critical for the performance of world generation and the prefab spawning system. It is the bridge between the asset loading pipeline and the live world state modification layer.

### Lifecycle & Ownership
The lifecycle of a **PrefabBuffer** is tightly controlled and requires manual memory management due to its direct use of Netty's **ByteBuf**. Failure to adhere to the lifecycle will result in memory leaks.

-   **Creation:** A **PrefabBuffer** is instantiated exclusively through its inner **PrefabBuffer.Builder**. The builder is typically invoked by a higher-level service, such as a **PrefabLoader**, which parses an asset file and translates it into a series of calls to **builder.addColumn**. The process culminates with a call to **builder.build()**, which finalizes the internal **ByteBuf** and returns the immutable **PrefabBuffer** instance.

-   **Scope:** A **PrefabBuffer** instance is intended to be long-lived. It is typically cached by a central prefab management service for the duration of a server session. The object itself is a lightweight container for the anchor, bounds, and the underlying **ByteBuf**. The actual data resides in off-heap memory managed by the **ByteBuf**.

-   **Destruction:** The **PrefabBuffer** must be explicitly destroyed by calling its **release** method. This deallocates the underlying **ByteBuf**. The owner of the **PrefabBuffer** (e.g., the caching service) is responsible for calling **release** when the prefab is no longer needed, typically during server shutdown or asset unloading. Similarly, each **PrefabBufferAccessor** must also be released after use.

## Internal State & Concurrency
-   **State:** The **PrefabBuffer** is immutable after its construction. All its fields, including the collections and vectors, are populated once by the **Builder** and are not modified thereafter. The only mutable aspect is the lifecycle state managed by the **release** method, which nullifies the internal buffer reference, transitioning the object to a terminal, unusable state.

-   **Thread Safety:** The **PrefabBuffer** is designed for concurrent reads but is not safe for concurrent modification.
    -   **Read Safety:** It is safe to have multiple threads create and use their own **PrefabBufferAccessor** instances on the same **PrefabBuffer** simultaneously. Each call to **newAccess** creates a retained duplicate of the buffer, giving each accessor its own independent reader index.
    -   **Write Safety:** The **Builder** is not thread-safe and must be confined to a single thread during the construction phase. Calling **release** on a **PrefabBuffer** while another thread is attempting to create an accessor or read from it will result in an **IllegalStateException** or undefined behavior.

## API Surface
The primary interaction points are the static builder, the accessor factory, and the release method. The main logic is exposed through the **PrefabBufferAccessor**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newBuilder() | PrefabBuffer.Builder | O(1) | Static factory to create a new builder instance. |
| newAccess() | PrefabBufferAccessor | O(1) | Creates a new, short-lived accessor for reading the buffer data. |
| release() | void | O(1) | **CRITICAL:** Releases the underlying ByteBuf. Must be called to prevent memory leaks. |
| PrefabBufferAccessor.forEach(...) | void | O(N) | The primary iteration method. Traverses all columns and blocks, invoking consumers. N is the total number of blocks. |
| PrefabBufferAccessor.getBlockId(x, y, z) | int | O(H) | Provides random access to a single block ID. Less performant than forEach as it must scan the column. H is the height of the column at (x, z). |
| PrefabBufferAccessor.release() | void | O(1) | **CRITICAL:** Releases the accessor's duplicated ByteBuf. Must be called after use. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve a cached **PrefabBuffer**, create a short-lived **PrefabBufferAccessor** to perform a world modification operation, and then immediately release the accessor.

**WARNING:** The accessor does not implement **AutoCloseable**. You must manually ensure **release** is called, preferably in a **finally** block.

```java
// 1. Retrieve a cached PrefabBuffer from a manager service
PrefabBuffer prefab = prefabManager.get("structures/tower.prefab");

// 2. Create a short-lived accessor to read the data
PrefabBuffer.PrefabBufferAccessor accessor = prefab.newAccess();
try {
    // 3. Create a call object to pass state to the consumers
    PrefabBufferCall callContext = new PrefabBufferCall(world, rotation, random);

    // 4. Use the high-performance forEach iterator to place the prefab
    accessor.forEach(
        (x, z, blockCount, ctx) -> { /* Column Predicate */ return true; },
        (x, y, z, blockId, ...) -> world.setBlock(x, y, z, blockId), // Block Consumer
        (x, z, entities, ctx) -> world.spawnEntities(x, z, entities), // Entity Consumer
        null, // Child Consumer
        callContext
    );
} finally {
    // 5. CRITICAL: Release the accessor to prevent memory leaks
    accessor.release();
}
```

### Anti-Patterns (Do NOT do this)
-   **Forgetting to Release:** Failing to call **release()** on either the **PrefabBuffer** or the **PrefabBufferAccessor** is a severe memory leak. The off-heap **ByteBuf** will not be garbage collected.
-   **Long-Lived Accessors:** Do not cache or hold references to **PrefabBufferAccessor** instances. They are designed to be created, used, and destroyed within a single operation.
-   **Using After Release:** Accessing a buffer or an accessor after its **release** method has been called will throw an **IllegalStateException**.
-   **Inefficient Access:** Avoid calling **getBlockId** or other random-access methods inside a tight loop. The **forEach** iterator is significantly more performant as it is a single forward-only pass over the serialized data.

## Data Pipeline
The **PrefabBuffer** serves as a compiled, intermediate representation in the prefab spawning pipeline.

> Flow:
> Prefab Asset File -> PrefabLoader Service -> **PrefabBuffer.Builder** -> **PrefabBuffer** (Cached) -> PrefabSpawner Service -> **PrefabBufferAccessor** -> World ChunkStore Modification


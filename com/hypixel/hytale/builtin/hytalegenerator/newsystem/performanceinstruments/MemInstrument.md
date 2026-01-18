---
description: Architectural reference for MemInstrument
---

# MemInstrument

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.performanceinstruments
**Type:** Interface

## Definition
```java
// Signature
public interface MemInstrument {
```

## Architecture & Concepts
The MemInstrument interface establishes a formal contract for any component within the world generation system that needs to report its own memory footprint. It acts as a standardized probe, allowing a centralized performance monitoring system to query disparate data structures for their memory usage without needing to know their internal implementation details.

This decouples the high-level performance aggregation logic from the low-level memory calculation of specific objects. By implementing this interface, a class signals its participation in the engine's memory accounting system. The interface also provides a set of static constants representing the known sizes of primitive types and common engine objects (e.g., VECTOR3I_SIZE, HASHMAP_ENTRY_SIZE). These constants ensure that memory calculations are consistent and accurate across all implementations.

This pattern is critical for preventing memory over-allocation and for debugging memory-intensive stages of procedural generation, where millions of objects may be created and destroyed in short periods.

### Lifecycle & Ownership
As an interface, MemInstrument itself does not have a lifecycle. It is a compile-time contract. The lifecycle of its *implementations* is dictated by the objects they are instrumenting.

- **Creation:** An object implementing MemInstrument is typically created at the same time as the data structure it monitors. For example, a hypothetical ChunkData class would create its corresponding ChunkMemInstrument upon its own instantiation.
- **Scope:** The scope of an implementation is tightly bound to the object it represents. It lives and dies with its owner.
- **Destruction:** The implementation is eligible for garbage collection when its owning object is destroyed and all references to it are released. There is no explicit destruction method defined in the contract.

## Internal State & Concurrency
- **State:** The interface is stateless. Implementations are expected to be stateless in the sense that a call to getMemoryUsage should not mutate the underlying object. The method is a read-only query.
- **Thread Safety:** The contract implicitly requires that implementations of getMemoryUsage are thread-safe. Performance data is often collected by a dedicated monitoring thread that runs concurrently with the main game logic and world generation threads. Implementations must not cause race conditions and should use appropriate synchronization if they need to read from mutable, shared data structures.

**WARNING:** Implementations of getMemoryUsage must be non-blocking and computationally inexpensive. A slow implementation can stall the entire performance monitoring system, leading to inaccurate metrics or engine-wide stutter.

## API Surface
The public contract consists of a single method and a nested record for the return value.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMemoryUsage() | MemInstrument.Report | Varies | Calculates and returns the total memory footprint in bytes of the associated object. Complexity depends on the implementation but should strive for O(1). |
| Report(long) | Record | O(1) | A simple, immutable data carrier for the memory usage report. Contains a single field, size_bytes. |

## Integration Patterns

### Standard Usage
The primary pattern is for a monitoring or management system to hold a collection of MemInstrument instances. It periodically iterates this collection to aggregate total memory usage.

```java
// A monitoring system polls all registered instruments
long totalMemoryUsage = 0L;
List<MemInstrument> instruments = getRegisteredInstruments();

for (MemInstrument instrument : instruments) {
    MemInstrument.Report report = instrument.getMemoryUsage();
    totalMemoryUsage += report.size_bytes();
}

System.out.println("Total instrumented memory: " + totalMemoryUsage + " bytes");
```

### Anti-Patterns (Do NOT do this)
- **Expensive Calculation:** Do not perform disk I/O, network requests, or deep recursive traversals within the getMemoryUsage method. This method must return quickly.
- **Inaccurate Reporting:** Do not hard-code values or omit significant internal data structures from the calculation. The purpose of this interface is accurate accounting. Use the provided static constants for consistency.
- **State Mutation:** The getMemoryUsage method must not alter the state of the object it is instrumenting. It is a read-only operation.

## Data Pipeline
MemInstrument is not part of a data processing pipeline but rather a query-response system.

> Query Flow:
> Performance Monitor -> `getMemoryUsage()` Call -> **MemInstrument Implementation** -> Memory Calculation -> `MemInstrument.Report` -> Performance Monitor Aggregator


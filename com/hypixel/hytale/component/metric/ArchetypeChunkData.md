---
description: Architectural reference for ArchetypeChunkData
---

# ArchetypeChunkData

**Package:** com.hypixel.hytale.component.metric
**Type:** Data Model / DTO

## Definition
```java
// Signature
public class ArchetypeChunkData {
```

## Architecture & Concepts
ArchetypeChunkData is a plain data structure that serves as a Data Transfer Object (DTO) for entity metrics. It is not an active component but rather a data container used to report performance or debugging information about the Entity Component System (ECS).

Within Hytale's ECS, an archetype is a unique combination of component types. This class encapsulates a summary for a group of entities that all share the same archetype. It holds two key pieces of information:
1.  **componentTypes:** The set of component names that define the archetype.
2.  **entityCount:** The number of entities that conform to this specific archetype within a given measurement scope (e.g., a world chunk or a server region).

The most critical feature of this class is the static **CODEC** field. This exposes a serialization and deserialization contract, allowing instances of ArchetypeChunkData to be efficiently encoded for network transmission or persisted to disk. This design indicates the class is a fundamental part of a data pipeline for engine diagnostics, server monitoring, or world data analysis.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-demand by higher-level systems, such as a MetricCollector or a WorldSerializer. They are not managed by a dependency injection context or service registry.
-   **Scope:** Transient. The lifetime of an ArchetypeChunkData object is extremely short. It exists only for the duration of a specific operation, such as being serialized into a network packet or aggregated into a larger report.
-   **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup as soon as all references to it are dropped. No manual destruction or cleanup methods are required.

## Internal State & Concurrency
-   **State:** **Mutable**. The class is designed to be instantiated (often with the default constructor by the codec) and then populated with data. Its internal fields are not final.
-   **Thread Safety:** **Not thread-safe**. This class contains no synchronization primitives and is not designed for concurrent access. It should be confined to a single thread or, if passed between threads, its state should not be modified after the hand-off (effectively treating it as immutable).

## API Surface
The public contract is minimal, consisting of simple data accessors. The static CODEC field is the primary integration point for serialization frameworks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentTypes() | String[] | O(1) | Returns the array of component type names that define the archetype. |
| getEntityCount() | int | O(1) | Returns the total count of entities matching this archetype. |

## Integration Patterns

### Standard Usage
The class is typically instantiated by a metric-gathering service, populated, and then passed to a serialization system.

```java
// A metric collection system creates an instance to report data.
String[] types = new String[]{"Position", "Velocity", "Renderable"};
int count = 1024;

ArchetypeChunkData report = new ArchetypeChunkData(types, count);

// The report is then passed to a serializer or logging service.
MetricService.submit(report);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation After Handoff:** Do not modify an ArchetypeChunkData instance after it has been passed to another system (e.g., a logging queue or network buffer). This can lead to race conditions and inconsistent data, as the receiving system may process the object at a later time.
-   **Use as a Map Key:** Because the object is mutable, using it as a key in a HashMap or HashSet is dangerous. If its internal state changes, its hash code will change, and the object may become unretrievable from the collection.

## Data Pipeline
ArchetypeChunkData acts as a data packet within a larger metrics or serialization pipeline. It does not process data itself; rather, it is the data being processed.

> Flow:
> Metric Collection Service -> **ArchetypeChunkData (Instantiation)** -> Serialization Engine (using CODEC) -> Network Stream or Log File


---
description: Architectural reference for EntityPlacementData
---

# EntityPlacementData

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.entity
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class EntityPlacementData implements MemInstrument {
```

## Architecture & Concepts
EntityPlacementData is an immutable Data Transfer Object (DTO) that serves as a blueprint for instantiating an entity within the world. It is a core component of the procedural world generation pipeline, specifically within the `hytalegenerator` system.

This class acts as a message, encapsulating the complete set of instructions required to place a single entity. It decouples the high-level *decision-making* logic of a generator (which decides *what* to place and *where*) from the low-level *execution* logic of the world storage system (which performs the actual placement).

By bundling the relative position (offset), orientation (rotation), entity type (entityHolder), and a unique identifier (objectId) into a single, immutable package, the system can safely queue and process thousands of these placement requests in parallel or batch operations without risk of data corruption. The implementation of the MemInstrument interface signifies its role in performance-critical code paths where memory allocation is closely monitored.

### Lifecycle & Ownership
- **Creation:** Instantiated by high-level world generation systems or prop applicators during the procedural generation of a world chunk or region. It is created when a generator determines an entity should exist at a specific location.
- **Scope:** Extremely short-lived and transient. An instance of EntityPlacementData exists only for the duration of a single generation task. It is created, passed to a processing queue, consumed by a world writer, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java Garbage Collector. It is destroyed once the entity has been placed in the world and no systems retain a reference to the placement instruction.

## Internal State & Concurrency
- **State:** **Immutable**. All member fields are declared final and are exclusively set during object construction. Once an EntityPlacementData object is created, its state cannot be altered. This design is critical for ensuring predictable and deterministic world generation.

- **Thread Safety:** **Inherently thread-safe**. Due to its immutability, an instance of EntityPlacementData can be safely shared and read by multiple threads without any external locking or synchronization. This allows a producer thread to generate placement data and pass it to a pool of consumer threads that write to the world state.

## API Surface
The public API consists entirely of simple accessors for its internal state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOffset() | Vector3i | O(1) | Returns the entity's position offset relative to a generation origin. |
| getRotation() | PrefabRotation | O(1) | Returns the entity's orientation. |
| getEntityHolder() | Holder<EntityStore> | O(1) | Returns a handle to the entity's prefab data. |
| getObjectId() | int | O(1) | Returns the unique identifier for this specific placement operation. |
| getMemoryUsage() | MemInstrument.Report | O(1) | Reports the estimated memory footprint of this object for performance instrumentation. |

## Integration Patterns

### Standard Usage
EntityPlacementData is intended to be created by a generator and immediately submitted to a processing system, such as a placement queue or a world transaction buffer.

```java
// Within a world generator or prop placement system
Vector3i calculatedOffset = new Vector3i(10, 64, -5);
PrefabRotation orientation = PrefabRotation.NORTH;
Holder<EntityStore> entityPrefab = getPrefabByName("boar");
int uniqueId = worldGenerationContext.generateUniqueId();

// Create the immutable placement instruction
EntityPlacementData placement = new EntityPlacementData(calculatedOffset, orientation, entityPrefab, uniqueId);

// Submit the instruction to the world modification queue for processing
worldGenerationContext.getPlacementQueue().add(placement);
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify the state of the objects returned by the accessors, for example, by calling `placement.getOffset().setX(5)`. While the EntityPlacementData object itself is immutable, the objects it references may not be. Modifying them breaks the "instruction" contract and can lead to severe, difficult-to-debug generation errors, especially in a multithreaded environment. Treat the returned objects as read-only.
- **Long-Term Storage:** Do not cache or store EntityPlacementData instances beyond the scope of a single generation task. They are transient data carriers, not persistent game state. Retaining them will cause a memory leak.

## Data Pipeline
This class is a fundamental data packet that flows from generation logic to world state modification.

> Flow:
> Prop Generation Logic -> **EntityPlacementData (Instantiation)** -> World Generation Placement Queue -> World Writer Service -> Entity Instantiated in EntityStore


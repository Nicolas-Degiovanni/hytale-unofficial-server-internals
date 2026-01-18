---
description: Architectural reference for EventSlotMapper
---

# EventSlotMapper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient State Object

## Definition
```java
// Signature
public class EventSlotMapper<EventType extends Enum<EventType>> {
```

## Architecture & Concepts
The EventSlotMapper is a build-time utility class responsible for converting a composite key, consisting of an enumeration type and an integer ID, into a single, unique, and contiguous integer slot. This process is a critical optimization for runtime systems, particularly in NPC behavior and animation state machines.

Its primary function is to "flatten" complex event identifiers. For example, instead of the runtime needing to process an event like (ON_TAKE_DAMAGE, variant 3), this class pre-calculates a simple integer, such as 5, to represent that unique combination. This allows runtime systems to use the resulting integer as a direct index into an array or a highly efficient key in a map, eliminating complex lookups and comparisons in performance-critical code.

This component operates exclusively during the asset compilation phase. It consumes raw configuration data and produces optimized mapping tables that are baked into the final server assets. It also aggregates metadata, such as the maximum range associated with any event that maps to a particular slot.

## Lifecycle & Ownership
- **Creation:** An EventSlotMapper is instantiated directly by a higher-level asset builder, such as one processing an NPC definition file. Its scope is temporary and tied to a single asset compilation task.
- **Scope:** The object lives only for the duration of a single asset's construction. It accumulates state as the asset's configuration is parsed. Once parsing is complete, its internal data structures are extracted, and the EventSlotMapper instance is intended to be discarded.
- **Destruction:** The object is eligible for garbage collection as soon as the asset builder that created it has retrieved the final mapping data and no longer holds a reference to the instance.

## Internal State & Concurrency
- **State:** The EventSlotMapper is highly mutable. Its core purpose is to build and accumulate state across multiple calls to its methods. The internal maps and the slot counter are modified on nearly every invocation of getEventSlot.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. The internal state, particularly the nextEventSlot counter and the unsynchronized map operations, will become corrupted if called from multiple threads without external locking. It is designed for single-threaded, sequential execution within an asset build pipeline.

## API Surface
The public API is minimal, focusing on the primary mutation method and getters to extract the final compiled data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEventSlot(type, set, maxRange) | int | O(1) avg | Assigns a unique slot ID for the given type/set pair. This is the primary method for populating the mapper's state. It is not idempotent for a given slot; subsequent calls with a larger maxRange will update the stored range. |
| getEventSets() | Map | O(1) | Returns the map of event types to the sets of IDs encountered for each. |
| getEventSlotMappings() | Map | O(1) | Returns the map of event types to their specific set-to-slot mappings. |
| getEventSlotRanges() | Int2DoubleMap | O(1) | Returns the final mapping of unique slot IDs to their maximum associated range value. |
| getEventSlotCount() | int | O(1) | Returns the total number of unique slots that have been allocated. |

## Integration Patterns

### Standard Usage
The intended use is to create a new instance for each asset being built. The builder then iterates through the asset's configuration, calling getEventSlot for each event definition. Finally, it retrieves the generated maps to construct the optimized runtime data structure.

```java
// During an NPC asset build process
EventSlotMapper<NpcBehaviorEvent> mapper = new EventSlotMapper<>(NpcBehaviorEvent.class, null);

// Loop through all behaviors defined in the NPC's configuration file
for (BehaviorConfig config : npcDefinition.getBehaviors()) {
    int slot = mapper.getEventSlot(config.getEventType(), config.getVariantId(), config.getRange());
    // Store this slot in a temporary build structure
    ...
}

// After parsing, extract the final data to bake into the runtime asset
NpcRuntimeAsset asset = new NpcRuntimeAsset();
asset.setEventSlotMappings(mapper.getEventSlotMappings());
asset.setEventSlotRanges(mapper.getEventSlotRanges());
asset.setSlotCount(mapper.getEventSlotCount());
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not re-use a single EventSlotMapper instance across the compilation of multiple, independent assets. This will cause slot IDs and state to leak from one asset to another, resulting in incorrect behavior and corrupted data.
- **Runtime Instantiation:** This class is a build-time tool. It should never be instantiated or used on a production game server during the main game loop. Its purpose is to *create* the data used by the runtime, not to be part of the runtime itself.
- **Concurrent Modification:** Never call getEventSlot from multiple threads on the same instance. This will lead to a race condition on the nextEventSlot counter and result in non-unique, incorrect slot assignments.

## Data Pipeline
The EventSlotMapper acts as a transformation stage within the server asset compilation pipeline. It does not originate data but rather reshapes it for runtime efficiency.

> Flow:
> NPC Definition File (JSON/HOCON) -> Asset Parser -> **EventSlotMapper** -> Final Runtime Data Structures (Baked into Asset)


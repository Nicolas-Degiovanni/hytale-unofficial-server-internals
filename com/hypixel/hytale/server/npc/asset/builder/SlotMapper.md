---
description: Architectural reference for SlotMapper
---

# SlotMapper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Utility

## Definition
```java
// Signature
public class SlotMapper {
```

## Architecture & Concepts
The SlotMapper is a stateful, special-purpose utility class designed to solve a common performance problem: the conversion of human-readable string identifiers into dense, zero-based integer indices. Within the server's NPC asset building pipeline, components are often defined by name in configuration files (e.g., "helmet", "main_hand", "chestplate"). At runtime, comparing and storing these strings is inefficient.

SlotMapper acts as a transient symbol table or an interning mechanism. For the duration of a single asset build process, it maintains a consistent, one-to-one mapping between a given string name and a unique integer slot. This allows the final, compiled asset to reference slots using highly efficient integer IDs, decoupling the runtime representation from the source configuration.

Its primary role is to provide stable, predictable integer IDs for a dynamic set of string names within a bounded context.

## Lifecycle & Ownership
- **Creation:** A SlotMapper is instantiated directly via its constructor (`new SlotMapper()`) by a higher-level process, typically an asset builder or a configuration parser. It is not managed by a dependency injection framework or a service registry.
- **Scope:** The lifecycle of a SlotMapper instance is intentionally short and is strictly tied to the parent operation that created it. It exists only for the duration of a single, cohesive task, such as building one specific NPC model's asset data.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the creating builder or process completes and the reference is dropped. It holds no static references and performs no background activity.

## Internal State & Concurrency
- **State:** The SlotMapper is a mutable, stateful object. Its internal maps, `mappings` and `nameMap`, are populated incrementally through calls to the `getSlot` method. The state represents a write-once cache; once a name is assigned a slot, that mapping is permanent for the lifetime of the object.
- **Thread Safety:** **WARNING: This class is NOT thread-safe.** The core `getSlot` method implements a non-atomic check-then-act pattern (`getInt`, check for `NO_SLOT`, then `put`). If multiple threads call `getSlot` concurrently for the same new name, a race condition will occur, leading to corrupted internal state, incorrect slot counts, and unpredictable behavior. All interactions with a single SlotMapper instance **must** be confined to a single thread or be protected by external synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSlot(String name) | int | O(1) | Resolves a string name to a stable integer slot. If the name has not been seen before, it allocates the next available slot and permanently maps the name to it. |
| slotCount() | int | O(1) | Returns the total number of unique names that have been mapped to slots. |
| getSlotMappings() | Object2IntMap | O(1) | Provides direct, read-only access to the internal name-to-slot mapping. Returns null if no slots have been created. |
| getNameMap() | Int2ObjectMap | O(1) | Provides direct, read-only access to the reverse slot-to-name mapping. Returns null if name tracking was disabled at construction. |

## Integration Patterns

### Standard Usage
The SlotMapper is designed to be used as a temporary helper within a larger build or serialization process. The typical pattern involves creating an instance at the start of the process, using it to resolve all necessary names, and then retrieving the final maps to embed into a runtime data structure.

```java
// A builder class processes a list of named equipment slots
SlotMapper equipmentMapper = new SlotMapper(true); // Enable name tracking for debugging
List<String> slotNamesFromConfig = List.of("head", "chest", "legs", "feet", "main_hand");

for (String name : slotNamesFromConfig) {
    int slotId = equipmentMapper.getSlot(name);
    // ... use the slotId to build part of an asset
}

// At the end of the process, retrieve the complete maps
Object2IntMap<String> finalMappings = equipmentMapper.getSlotMappings();
// ... serialize finalMappings into the game asset
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Access:** Never share a single SlotMapper instance across multiple threads without external locking. The resulting race conditions will corrupt the mappings and lead to difficult-to-diagnose bugs.
- **Instance Reuse:** Do not reuse a SlotMapper instance for logically separate build processes. For example, building an Orc asset and a Human asset should use two distinct SlotMapper instances. Reusing an instance will cause slot ID collisions and merge the symbol tables of unrelated assets.
- **Long-Term Storage:** Do not hold onto a SlotMapper instance beyond the scope of the build task. It is a transient tool, not a permanent runtime service. The maps it generates should be extracted and stored in a more appropriate read-only runtime structure.

## Data Pipeline
The SlotMapper functions as a transformer, converting a stream of string identifiers into a structured, indexed map.

> Flow:
> Configuration Source (e.g., JSON file) -> Asset Builder Logic -> **SlotMapper.getSlot()** -> Integer Slot ID -> Final Runtime Asset


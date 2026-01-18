---
description: Architectural reference for ReferenceSlotMapper
---

# ReferenceSlotMapper<T>

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class ReferenceSlotMapper<T> extends SlotMapper {
```

## Architecture & Concepts
The ReferenceSlotMapper is a specialized, stateful utility class that functions as a lazy, memoized factory for objects. Its primary role is to bridge the gap between symbolic string identifiers, often defined in asset or configuration files, and their corresponding live object instances within the server.

It builds upon the functionality of its parent, SlotMapper, which handles the core mapping of a unique string name to a stable integer slot. ReferenceSlotMapper extends this by maintaining a list of objects, indexed by these integer slots. When a reference for a name is requested, it first resolves the name to its integer slot. If an object for that slot has already been created, it is returned. If not, a new object is instantiated on-demand using a provided factory (a Supplier), stored in the list at the correct slot, and then returned.

This pattern is critical within asset building pipelines, such as for NPC or entity definitions. It ensures that multiple references to the same logical component (e.g., an attachment point named "head_slot") within a single definition file resolve to the *exact same object instance*, preventing redundant allocations and ensuring data consistency.

## Lifecycle & Ownership
- **Creation:** An instance of ReferenceSlotMapper is typically created at the beginning of a specific, bounded asset-building process. For example, a new mapper would be instantiated by an NpcDefinitionBuilder each time it begins parsing a new NPC JSON file. It is not a global service and is not managed by a dependency injection framework.
- **Scope:** The object's lifetime is strictly limited to the scope of the single build operation it supports. It accumulates state as the asset is processed and is discarded once the final asset is constructed.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the parent builder that created it completes its task and goes out of scope. It holds no persistent resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its internal list of references grows dynamically as new, unique names are passed to the getReference method. The state of the parent SlotMapper, which maps names to integers, is also mutated concurrently.
- **Thread Safety:** **This class is not thread-safe.** The internal operations, particularly the check-then-add logic within the getReference method, are susceptible to race conditions. If multiple threads access the same instance to resolve new names simultaneously, it can lead to inconsistent state, incorrect list sizes, or runtime exceptions. All access to a shared instance must be externally synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReference(String name) | T | Amortized O(1) | Retrieves the object instance associated with the given name. If no instance exists, it is created using the configured supplier, cached, and returned. Subsequent calls with the same name will return the identical cached instance. |
| getReferenceList() | List<T> | O(1) | Returns a direct reference to the internal list of all created objects. **Warning:** Modifying this list externally will corrupt the internal state of the mapper. |

## Integration Patterns

### Standard Usage
The class is intended to be used within a builder or factory to manage the resolution of named sub-components during a larger object's construction.

```java
// Example: A builder for a character model uses a mapper to
// resolve string-based bone names to AttachmentPoint objects.

Supplier<AttachmentPoint> pointFactory = () -> new AttachmentPoint();
ReferenceSlotMapper<AttachmentPoint> boneMapper = new ReferenceSlotMapper<>(pointFactory);

// The first request for "hand_r" creates a new AttachmentPoint.
AttachmentPoint rightHand = boneMapper.getReference("hand_r");
rightHand.setOffset(1.0, 0.0, 0.0);

// A subsequent request for the same name returns the *exact same instance*.
AttachmentPoint sameRightHand = boneMapper.getReference("hand_r");

// The final list of unique points can be retrieved for the model.
List<AttachmentPoint> allPoints = boneMapper.getReferenceList();
```

### Anti-Patterns (Do NOT do this)
- **Multi-threaded Access:** Do not share a single ReferenceSlotMapper instance across multiple threads without implementing external locking. The internal state will become corrupted.
- **External State Mutation:** Do not modify the list returned by getReferenceList. This list is the internal state of the mapper, and external additions or removals will break the mapping logic, leading to unpredictable behavior or IndexOutOfBoundsExceptions.
- **Long-Lived Scope:** Do not maintain instances of ReferenceSlotMapper in a long-lived or global scope. They are designed as short-lived, single-use helpers for a specific task and will hold references to all created objects, preventing them from being garbage collected.

## Data Pipeline
The ReferenceSlotMapper acts as a stateful transformer within a data processing flow, typically during asset loading.

> Flow:
> Symbolic Name (String) from Config File -> **ReferenceSlotMapper.getReference()** -> [Internal: SlotMapper resolves String to int] -> [Internal: Checks list for existing object] -> [Internal, if needed: Supplier<T> creates new object] -> Concrete Object Instance (T) -> Returned to Asset Builder


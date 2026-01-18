---
description: Architectural reference for PersistentParameter
---

# PersistentParameter

**Package:** com.hypixel.hytale.server.npc.storage
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class PersistentParameter<Type> {
```

## Architecture & Concepts
The PersistentParameter class is an abstract base class that forms a critical link between an entity's state and the server's world persistence mechanism. It is not a concrete data storage class itself, but rather a template for creating parameters that, when modified, automatically signal to the engine that the owning entity's data has changed and needs to be saved.

Its core architectural function is to encapsulate the side-effect of "dirtying" a world chunk. Any change to a subclass of PersistentParameter guarantees that the `TransformComponent` of the owning entity is notified. This notification flags the chunk containing the entity as modified, queuing it for serialization and storage to disk by the `EntityStore` and world saving systems. This pattern decouples game logic (e.g., changing an NPC's home location) from the low-level mechanics of world persistence.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of PersistentParameter are instantiated as member fields within higher-level server components, typically those associated with NPC state. They are not managed by a dependency injection framework or service locator.
- **Scope:** The lifetime of a PersistentParameter instance is strictly tied to its owning component. It exists as long as the entity it belongs to is loaded in memory.
- **Destruction:** The object is eligible for garbage collection when its owning component and entity are unloaded or destroyed. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The actual value of the parameter is stored within the concrete subclass implementation. The state is, by design, mutable. This base class itself is stateless.
- **Thread Safety:** This class is **not thread-safe**. The `set` method performs component lookups and modifications via a `ComponentAccessor`. All interactions with the component system must be performed on the main server thread to prevent race conditions and data corruption. External synchronization is required if modification from other threads is absolutely necessary, though this is strongly discouraged.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(ownerRef, value, accessor) | void | O(1) | Sets the parameter's value and marks the owner's chunk as dirty, triggering persistence. This is the primary entry point for modification. |

## Integration Patterns

### Standard Usage
The intended pattern is to define a concrete subclass and embed it within a component. Game logic then interacts with the `set` method to modify state, relying on the class to handle the persistence trigger.

```java
// Within a game system or component method...

// Assume myNpcRef is a valid Ref<EntityStore> for an NPC
// Assume myComponentAccessor is the active accessor for the current tick

// Retrieve the NPC's state component
NpcStateComponent state = myComponentAccessor.getComponent(myNpcRef, NpcStateComponent.getComponentType());

if (state != null) {
    // The 'name' field would be a concrete PersistentParameter<String>
    state.name.set(myNpcRef, "Guard Captain", myComponentAccessor);
}
```

### Anti-Patterns (Do NOT do this)
- **Bypassing the `set` Method:** Directly calling the protected `set0` method (if exposed by a subclass) will update the value in memory but **will not** mark the chunk as dirty. This will result in the change being lost upon server restart or chunk unload.
- **Use for Transient Data:** Do not use PersistentParameter for state that does not need to be saved (e.g., temporary AI target references). The overhead of the `markChunkDirty` operation is unnecessary and inefficient for such data.
- **Cross-Thread Modification:** Calling `set` from any thread other than the main server thread without proper, engine-aware synchronization will lead to severe concurrency issues within the component system.

## Data Pipeline
The primary data flow is outbound, from game logic to the persistence layer. It acts as a trigger, not a complex data processor.

> Flow:
> Game Logic Event -> `PersistentParameter.set()` -> Internal state update via `set0()` -> `TransformComponent.markChunkDirty()` -> World Storage System -> Chunk Serialization -> Disk


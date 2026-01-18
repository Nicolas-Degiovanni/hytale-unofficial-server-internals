---
description: Architectural reference for PersistentRef
---

# PersistentRef

**Package:** com.hypixel.hytale.server.core.entity.reference
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class PersistentRef {
```

## Architecture & Concepts
The PersistentRef class is a smart reference system designed to bridge the gap between a persistent, serializable entity identity (a UUID) and its transient, in-memory representation (a Ref<EntityStore>). Its primary function is to allow game components to maintain stable references to other entities that can survive server restarts, chunk unloading/loading cycles, and network serialization.

This class is a cornerstone of the server's data integrity model. Instead of holding a direct memory pointer, which would become invalid when an entity is unloaded, a component holds a PersistentRef. This object stores the target entity's globally unique UUID.

At runtime, when a component needs to interact with its target, it uses the PersistentRef to resolve the UUID back into a live, in-memory entity reference. This resolution is performed on-demand (lazy-loaded), preventing the system from having to load entire chains of referenced entities into memory simultaneously. The internal Ref<EntityStore> acts as a temporary cache for this resolved reference, providing fast access for subsequent calls within the same game tick.

The presence of a static CODEC field signifies that this class is deeply integrated with the server's serialization framework, enabling world state to be saved and loaded reliably.

## Lifecycle & Ownership
-   **Creation:** A PersistentRef is not a standalone service. It is typically instantiated as a field within another game Component. Its creation is tied to the creation of its owning component, either through direct instantiation or during deserialization from world storage.
-   **Scope:** The lifetime of a PersistentRef instance is strictly bound to its owning component. It exists as long as its parent object exists.
-   **Destruction:** The object is reclaimed by the Java Garbage Collector when its owning component is destroyed and no longer referenced. The `clear` method can be used to manually sever the link and invalidate the reference, but this does not destroy the PersistentRef object itself.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable. The core fields, `uuid` and `reference`, are frequently modified by API calls. The `reference` field serves as a write-through cache for the UUID-to-Ref lookup, which is populated by the `getEntity` method and invalidated by `setUuid` or `clear`.

-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. Concurrent access from multiple threads can lead to race conditions and an inconsistent internal state. For example, one thread calling `clear` while another is in the middle of the caching logic within `getEntity` can produce unpredictable behavior.

    **WARNING:** All operations on a PersistentRef instance must be performed from a single, controlled thread, typically the main server thread responsible for the entity's world simulation. Do not share instances across asynchronous tasks without explicit external locking.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setEntity(Ref, ComponentAccessor) | void | O(1) + Component Lookup | Establishes a link to a live entity, extracting and storing its UUID. |
| getEntity(ComponentAccessor) | Ref<EntityStore> | O(1) to O(log N) | Resolves the stored UUID into a live entity reference. O(1) on cache hit; O(log N) or higher on cache miss, depending on the world's entity lookup complexity. |
| clear() | void | O(1) | Invalidates the reference by nullifying both the UUID and the cached entity reference. |
| isValid() | boolean | O(1) | Checks for the presence of a non-null UUID. This is the primary check for whether the reference is configured. |

## Integration Patterns

### Standard Usage
A component uses PersistentRef to maintain a link to another entity. To interact with the target, it must first resolve the reference using `getEntity`. This pattern is mandatory for all cross-entity references that must persist.

```java
// Example: A turret component targeting a player
public class TurretComponent extends Component {
    private PersistentRef currentTarget;

    public void acquireTarget(Ref<EntityStore> targetEntity, ComponentAccessor accessor) {
        this.currentTarget.setEntity(targetEntity, accessor);
    }

    public void update(ComponentAccessor accessor) {
        if (this.currentTarget.isValid()) {
            Ref<EntityStore> target = this.currentTarget.getEntity(accessor);
            // The resolved target can be null if the entity was removed from the world
            if (target != null && target.isValid()) {
                // Fire at the target's position
            } else {
                // Target is no longer valid, clear it
                this.currentTarget.clear();
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Caching of Resolved References:** Do not call `getEntity` once and store the resulting `Ref<EntityStore>` in a field for long-term use. The target entity can be unloaded at any time, invalidating the stored `Ref`. The purpose of PersistentRef is to handle this re-resolution safely. **Always re-call `getEntity` when you need to access the entity.**

-   **Ignoring Validity Checks:** Calling `getEntity` without first checking `isValid()` is inefficient and poor practice. If no UUID is set, the call will immediately return null, but the initial check makes intent clear and avoids the method call overhead.

-   **Cross-Thread Modification:** Never modify a PersistentRef from an asynchronous task or a different thread than the one that owns the component. This will lead to data corruption and severe concurrency bugs.

## Data Pipeline
The PersistentRef acts as a critical node in the data serialization and resolution pipeline.

**Serialization (World Save)**
> Flow:
> Owning Component -> **PersistentRef** -> `uuid` field extraction -> `CODEC` -> Binary UUID -> World Storage

**Deserialization & Resolution (World Load & Gameplay)**
> Flow:
> World Storage -> Binary UUID -> `CODEC` -> **PersistentRef** (with `uuid` populated) -> Runtime call to `getEntity()` -> `ComponentAccessor` lookup -> Resolved `Ref<EntityStore>`


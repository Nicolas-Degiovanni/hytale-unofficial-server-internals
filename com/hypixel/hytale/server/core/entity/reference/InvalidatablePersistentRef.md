---
description: Architectural reference for InvalidatablePersistentRef
---

# InvalidatablePersistentRef

**Package:** com.hypixel.hytale.server.core.entity.reference
**Type:** Data Object

## Definition
```java
// Signature
public class InvalidatablePersistentRef extends PersistentRef {
```

## Architecture & Concepts
The InvalidatablePersistentRef is a specialized "smart pointer" for server-side entities. It extends the functionality of PersistentRef, which provides a stable, serializable reference to an entity. Its primary architectural purpose is to solve the "stale reference" problem in a persistent world where entities can be destroyed and their IDs potentially recycled.

This class introduces a versioning mechanism through a reference counter. When an InvalidatablePersistentRef is bound to an entity, it not only stores the entity's unique ID but also captures the current value of the entity's **PersistentRefCount** component.

The core of its design is the `validateEntityReference` method. Before an entity reference is used, this method is called to ensure its integrity. Validation fails if:
1.  The base PersistentRef validation fails (e.g., the entity no longer exists).
2.  The stored reference count within this object does not match the *current* reference count on the target entity's PersistentRefCount component.

This mechanism allows the engine to perform significant lifecycle operations on an entity (e.g., unloading and reloading it, which might increment its PersistentRefCount) and reliably invalidate any old handles pointing to its previous state. It is a critical component for maintaining data integrity and preventing bugs related to use-after-invalidation.

### Lifecycle & Ownership
-   **Creation:** An instance is created either programmatically by game logic needing to point to an entity, or automatically by the serialization framework using the provided static CODEC during deserialization from storage or network streams.
-   **Scope:** The lifetime of an InvalidatablePersistentRef is tied to its containing object, typically another component. It does not manage its own lifecycle.
-   **Destruction:** The object is eligible for garbage collection when its owner is destroyed and no longer holds a reference to it. The `clear` method can be called to explicitly nullify its internal state, invalidating it immediately.

## Internal State & Concurrency
-   **State:** This class is mutable. It holds the entity reference (inherited from PersistentRef) and an integer, `refCount`. This `refCount` is a snapshot of the target entity's state at the moment of binding via `setEntity`.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. All method calls that read or modify its state must be synchronized externally, typically by ensuring all access occurs on the main server world thread. Unsynchronized access from multiple threads will lead to race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setEntity(ref, accessor) | void | O(1) | Binds this reference to a target entity. Reads the entity's PersistentRefCount component and stores its value internally. |
| clear() | void | O(1) | Resets the reference and sets the internal refCount to -1, effectively invalidating it. |
| getRefCount() | int | O(1) | Returns the reference count value that was captured when this object was bound. |
| validateEntityReference(ref, accessor) | boolean | O(1) | Performs integrity check. Returns true only if the base reference is valid and the stored refCount matches the target entity's current refCount. |

## Integration Patterns

### Standard Usage
This object is intended to be held as a field within a component that needs a long-lived, verifiable reference to another entity. Before dereferencing the pointer, the system must validate it.

```java
// A component holding a reference to a target
public class MyComponent extends Component {
    private InvalidatablePersistentRef targetRef = new InvalidatablePersistentRef();

    // When setting the target
    public void setTarget(Ref<EntityStore> entityRef, ComponentAccessor<EntityStore> accessor) {
        this.targetRef.setEntity(entityRef, accessor);
    }

    // When using the target
    public void doSomethingWithTarget(ComponentAccessor<EntityStore> accessor) {
        Ref<EntityStore> resolvedRef = this.targetRef.get(accessor); // get() internally validates
        if (resolvedRef.isPresent()) {
            // Safely use the entity
        } else {
            // Handle the case where the reference is now invalid
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Storing a reference and using it later without re-validating it completely defeats the purpose of this class. Always assume the reference could have become stale.
-   **Manual State Management:** Do not call `setRefCount` directly. This method is exposed for the serialization CODEC. The reference count should only be set via a call to `setEntity`.
-   **Storing Resolved References:** Do not resolve the reference, store the resulting `Ref<EntityStore>`, and use it later. The resolved reference itself provides no invalidation guarantees; only the InvalidatablePersistentRef object does.

## Data Pipeline
This class acts as a stateful container within a data validation flow. It does not process data itself but provides the logic to verify data integrity.

> **Flow:**
>
> 1.  **Binding:** Game Logic -> `setEntity` -> **InvalidatablePersistentRef** reads and stores `refCount` from target Entity.
> 2.  **Serialization:** Engine -> `CODEC` -> **InvalidatablePersistentRef** state (ID and refCount) is written to a byte stream.
> 3.  **Deserialization:** Engine -> `CODEC` -> **InvalidatablePersistentRef** is reconstructed from the byte stream.
> 4.  **Validation:** Game Logic -> `validateEntityReference` -> **InvalidatablePersistentRef** reads the *current* `refCount` from the target Entity and compares it against its stored value.


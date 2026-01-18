---
description: Architectural reference for WeakComponentReference
---

# WeakComponentReference

**Package:** com.hypixel.hytale.component
**Type:** Reference Object

## Definition
```java
// Signature
public class WeakComponentReference<ECS_TYPE, T extends Component<ECS_TYPE>> {
```

## Architecture & Concepts
The WeakComponentReference is a smart pointer within Hytale's Entity-Component-System (ECS) architecture. Its primary function is to provide a non-owning, garbage-collectible handle to a Component instance. This design is critical for memory management in a dynamic game world where entities and their components are frequently created and destroyed.

This class solves two key problems:
1.  **Memory Leaks:** By using an internal Java WeakReference, it allows a Component to be garbage collected if no other part of the engine holds a strong reference to it. This prevents systems from accidentally keeping components in memory long after their parent entity has been destroyed.
2.  **Dangling Pointers:** It provides resilience against stale references. If the underlying Component object is garbage collected, the WeakComponentReference does not become invalid. Instead, it retains the stable identity of the entity (Ref) and the component type (ComponentType). A subsequent call to its get method will attempt to "rehydrate" the reference by querying the central component Store, transparently fetching a new, valid reference to the component if it still exists in the ECS world.

In essence, it acts as a lazy-loading, auto-recovering handle to an ECS component, making it safe to store and pass around without risking memory leaks or null pointer exceptions from destroyed objects.

### Lifecycle & Ownership
-   **Creation:** Instances are not intended for direct user creation. The package-private constructor indicates that they are created and managed exclusively by other core ECS classes, most notably the component Store, when a request for a weak handle to a component is made.
-   **Scope:** A WeakComponentReference object can be held for any duration. However, its ability to resolve to a valid Component is strictly tied to the lifetime of that component within the master Store.
-   **Destruction:** The object itself is subject to standard Java garbage collection when it is no longer referenced. More importantly, its internal link to the ECS world is severed when the package-private `invalidate` method is called, typically by the Store when an entity or component is explicitly removed. This action nullifies the entity reference, permanently preventing any future rehydration attempts.

## Internal State & Concurrency
-   **State:** This object is highly mutable. Its internal WeakReference can be cleared at any point by the Java Garbage Collector. The `get` method is non-pure, as it may modify the internal state by repopulating the WeakReference upon a successful re-fetch from the Store.
-   **Thread Safety:** **This class is not thread-safe.** The `get` method contains a check-then-act sequence that is vulnerable to race conditions. If multiple threads call `get` on the same instance after the weak reference has been cleared, they may perform redundant lookups in the Store and non-atomic writes to the internal `reference` field. All access to a shared WeakComponentReference instance from multiple threads must be externally synchronized.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | T | O(1) Amortized | Attempts to retrieve the component. Returns from a hot cache if available, otherwise re-queries the central Store. Returns null if the component has been permanently destroyed. |
| getStore() | Store | O(1) | Returns a reference to the master component Store this handle belongs to. |
| getType() | ComponentType | O(1) | Returns the type token for the component this handle targets. |
| getEntityReference() | Ref | O(1) | Returns the stable identifier of the entity that owns the component. May be null if invalidated. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve a WeakComponentReference from an authoritative source (like the ECS Store) and use its `get` method to resolve the component when needed, always accounting for the possibility of a null return value.

```java
// Assume 'store' is a valid Component Store
// and 'entityRef' points to an existing entity with a Position component.

WeakComponentReference<EcsId, Position> weakPosRef = store.getWeakComponentReference(entityRef, Position.class);

// ... time passes, other systems run, a GC may occur ...

// Always null-check the result of get()
Position currentPosition = weakPosRef.get();
if (currentPosition != null) {
    // Safely work with the component
    currentPosition.setX(100);
} else {
    // The entity or component was destroyed. Handle this case gracefully.
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never attempt to create an instance with `new`. The package-private constructor forbids this, and doing so would bypass the essential link to the central component Store, rendering the object useless.
-   **Long-Term Caching of `get()` Result:** Storing the object returned by `get()` in a long-lived field defeats the entire purpose of using a weak reference. This creates a new strong reference, preventing the component from being garbage collected and re-introducing the risk of memory leaks.
-   **Assuming Non-Null Return:** Never assume `get()` will return a non-null value. The component can be destroyed at any time between calls. Failure to null-check the result is a common source of NullPointerException errors.

## Data Pipeline
This class does not process data in a traditional pipeline. Instead, it represents a conditional data retrieval flow.

> **Flow (Cache Hit):**
> Caller -> `get()` -> **WeakComponentReference** (Internal WeakReference is valid) -> Return Component

> **Flow (Cache Miss / Rehydration):**
> Caller -> `get()` -> **WeakComponentReference** (Internal WeakReference is empty) -> `Store.getComponent()` -> **WeakComponentReference** (Repopulates internal reference) -> Return Component


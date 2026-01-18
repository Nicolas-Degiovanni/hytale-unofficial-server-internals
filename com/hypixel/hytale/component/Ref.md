---
description: Architectural reference for Ref
---

# Ref

**Package:** com.hypixel.hytale.component
**Type:** Transient

## Definition
```java
// Signature
public class Ref<ECS_TYPE> {
```

## Architecture & Concepts
The Ref class is a foundational primitive within Hytale's Entity-Component-System (ECS) architecture. It functions as a **stable, indirect handle** to a specific component instance, rather than a direct Java object reference. This design is critical for the performance and memory management of the engine.

By decoupling the identity of a component (the Ref) from its data (stored contiguously in a Store), the system gains several advantages:
- **Cache Locality:** Systems can iterate directly over the backing arrays in the Store, leading to highly efficient memory access patterns.
- **Indirection and Stability:** The actual component data can be moved in memory (e.g., during Store compaction) without invalidating existing Refs. Only the internal index of the Ref needs to be updated by the owning Store.
- **Explicit Lifecycle Management:** A Ref can be explicitly invalidated, preventing "use-after-free" bugs that are common with raw pointers or direct object references in complex lifecycles. The `invalidatedBy` field provides robust debugging information, capturing the stack trace of the exact moment a Ref becomes stale.

In essence, a Ref does not hold component data; it holds the *coordinates* to find that data within the authoritative Store.

### Lifecycle & Ownership
- **Creation:** A Ref is never created directly by game logic. It is exclusively instantiated and vended by a corresponding Store or a higher-level EntityManager when a new component is allocated. The Ref is returned to the caller as a handle to the newly created component.
- **Scope:** The Ref object itself is a lightweight, transient value object that can be freely passed between systems. However, its **validity** is strictly tied to the lifetime of the component data it points to within the Store. It remains valid from the moment of creation until the component is explicitly removed from the ECS world.
- **Destruction:** The Java object is reclaimed by the garbage collector when no longer referenced. Critically, the *validity* of the Ref is terminated via the package-private `invalidate` method, which is called by the owning Store when a component is deleted. This action sets the internal index to a sentinel value (`Integer.MIN_VALUE`) and makes the Ref unusable. Any subsequent attempt to use it will trigger an `IllegalStateException`.

## Internal State & Concurrency
- **State:** The state is mutable. While the reference to the `store` is final, the `index` is volatile and is expected to change, primarily during invalidation. The `hashCode` is a cached, volatile field to optimize performance in collections like HashMaps.
- **Thread Safety:** The class is designed to be conditionally thread-safe. The use of `volatile` for `index`, `hashCode`, and `invalidatedBy` ensures that changes to a Ref's validity are immediately visible across all threads. This makes it safe to read a Ref's state or check its validity from any thread.

**WARNING:** While the Ref handle itself is safe to pass and inspect across threads, this provides **no guarantee** about the thread safety of the underlying component data in the Store. All mutations of component data must be synchronized through appropriate engine-level mechanisms, such as command buffers or job system dependencies. The package-private access modifiers on `setIndex` and `invalidate` enforce that only the owning Store can manage the Ref's lifecycle, which is a key part of the concurrency control strategy.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getStore() | Store<ECS_TYPE> | O(1) | Returns the authoritative Store that owns the component data. |
| getIndex() | int | O(1) | Returns the current index of the component data within its Store. |
| isValid() | boolean | O(1) | Performs a non-throwing check to see if the Ref currently points to a valid component. |
| validate() | void | O(1) | Asserts that the Ref is valid. Throws an `IllegalStateException` if the Ref has been invalidated. |

## Integration Patterns

### Standard Usage
A Ref should always be treated as an opaque handle. Game systems receive a Ref and use it in conjunction with its Store to access or modify the actual component data. Validity should always be checked before use in contexts where the component might have been deleted.

```java
// Correctly accessing component data via a Ref
void processMovement(Ref<Position> positionRef) {
    if (!positionRef.isValid()) {
        // The entity was likely destroyed this tick, so we skip it.
        return;
    }
    
    Store<Position> positionStore = positionRef.getStore();
    Position pos = positionStore.get(positionRef.getIndex()); // Hypothetical Store API
    pos.x += 1.0f;
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new Ref()`. A default-constructed Ref is invalid and serves no purpose. Refs must be obtained from the ECS framework.
- **Ignoring Validity:** Failing to call `isValid()` or `validate()` before accessing the Store can lead to `ArrayIndexOutOfBoundsException` or, worse, the modification of data belonging to a completely different component that now occupies the old index.
- **Storing Refs Across Major State Changes:** Do not cache a Ref across long-running operations or major world state transitions (e.g., level loading) without re-validating it. The underlying component is likely to have been destroyed and the Ref invalidated.

## Data Pipeline
The Ref is not part of a data processing pipeline itself, but rather a core mechanism for data access within one. It acts as the lookup key to retrieve data from the primary ECS storage.

> Flow:
> System Logic -> Receives **Ref<Component>** -> Checks `isValid()` -> Accesses `getStore()` -> Uses `getIndex()` to retrieve Component Data -> Processes Data


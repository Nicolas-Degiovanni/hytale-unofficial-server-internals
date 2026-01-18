---
description: Architectural reference for CachedSupplier
---

# CachedSupplier

**Package:** com.hypixel.hytale.function.supplier
**Type:** Utility

## Definition
```java
// Signature
public class CachedSupplier<T> implements Supplier<T> {
```

## Architecture & Concepts
The CachedSupplier is a foundational concurrency utility designed for lazy initialization and memoization. It acts as a thread-safe wrapper around a delegate Supplier, ensuring that an expensive computation or resource acquisition is executed only once.

Its primary role in the engine is performance optimization. By deferring costly operations until they are absolutely necessary and caching the result, it significantly reduces redundant work. This pattern is critical for systems that frequently access shared, immutable resources whose creation is computationally intensive, such as procedural noise maps, compiled shader programs, or complex data models loaded from disk.

The implementation employs a **double-checked locking** pattern. This ensures minimal contention by performing a low-cost volatile read before acquiring a more expensive monitor lock, guaranteeing both performance in the common case (value is already initialized) and correctness in the rare case (multiple threads race to initialize the value).

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor, typically as a private final field within a service or component that requires a lazily-initialized resource. The creator is responsible for providing the delegate Supplier and retains full ownership of the CachedSupplier instance. It is not managed by a central registry.
- **Scope:** The lifecycle of a CachedSupplier is bound to the lifecycle of its owning object. It persists as long as its owner is in memory.
- **Destruction:** The object is eligible for garbage collection once its owner is no longer referenced. The `invalidate` method can be used to force re-computation on the next access, but it does not destroy the object itself.

## Internal State & Concurrency
- **State:** The class is stateful and mutable, transitioning from an *uninitialized* to an *initialized* state. Its internal fields, `value` and `initialized`, are marked as `transient` to prevent the cached state from being serialized. This is a deliberate design choice to ensure that deserialized instances re-run their expensive initialization logic rather than using potentially stale, deserialized data.
- **Thread Safety:** This class is guaranteed to be thread-safe. The `get` and `invalidate` methods are synchronized to prevent race conditions during state transitions. The use of a `volatile` boolean flag for the `initialized` state ensures that writes are immediately visible across all threads, which is essential for the correctness of the double-checked locking pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | T | O(N) first call, O(1) after | Retrieves the value. On the first call, it blocks to execute the delegate and caches the result. Subsequent calls return the cached value instantly. |
| getValue() | T | O(1) | Returns the currently cached value without triggering computation. **Warning:** May return null if `get` has not been called. |
| invalidate() | void | O(1) | Resets the supplier to its uninitialized state, clearing the cached value. The next call to `get` will re-execute the delegate. |

## Integration Patterns

### Standard Usage
The primary use case is to wrap an expensive, one-time operation within a class field, ensuring the operation only runs when the result is first needed.

```java
// Example: Lazily loading a complex game asset
public class WorldGenerator {
    // The delegate defines the expensive operation
    private final Supplier<NoiseMap> noiseMapLoader = () -> NoiseMap.generate(this.seed);

    // The CachedSupplier ensures the generation only happens once
    private final CachedSupplier<NoiseMap> cachedNoiseMap = new CachedSupplier<>(noiseMapLoader);

    public void generateChunk(int x, int z) {
        // The first call to this method will trigger NoiseMap.generate() and block.
        // All subsequent calls in any thread will receive the cached instance instantly.
        NoiseMap map = cachedNoiseMap.get();
        // ... use map to generate chunk
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Delegates with Side-Effects:** The delegate Supplier should be idempotent. Do not rely on side-effects within the delegate, as it is only guaranteed to execute once. Logic that depends on these side-effects will be unreliable.
- **Recursive Deadlocks:** Do not create a dependency graph where a delegate's `get` method can, through any chain of calls, attempt to call `get` on the same CachedSupplier instance. This will result in a deadlock as the thread will wait for a lock it already holds.
- **Unsafe Peeking:** Do not use `getValue` to check for initialization before calling `get`. This pattern is not thread-safe and defeats the purpose of the internal locking mechanism. Always call `get` when you need to ensure the value is present.

## Data Pipeline
CachedSupplier acts as a conditional gate or source in a data flow. It does not process a stream of data but rather materializes a value on demand.

> **First Call Flow:**
> `get()` -> **[CachedSupplier]** -> Check `initialized` (false) -> Acquire Lock -> Check `initialized` (false) -> `delegate.get()` -> Store Result -> Set `initialized` (true) -> Release Lock -> Return Result

> **Subsequent Call Flow:**
> `get()` -> **[CachedSupplier]** -> Check `initialized` (true) -> Return Cached Result


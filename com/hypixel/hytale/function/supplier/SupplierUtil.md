---
description: Architectural reference for SupplierUtil
---

# SupplierUtil

**Package:** com.hypixel.hytale.function.supplier
**Type:** Utility

## Definition
```java
// Signature
public class SupplierUtil {
```

## Architecture & Concepts
SupplierUtil is a stateless factory class that provides a centralized, standardized mechanism for creating memoized, lazily-evaluated suppliers. Its primary role in the engine's architecture is to promote performance optimization by simplifying the implementation of the lazy initialization and caching pattern.

By wrapping a standard Java Supplier, this utility produces a CachedSupplier instance. This ensures that an expensive or resource-intensive operation encapsulated within the original supplier is executed exactly once. All subsequent requests for the value retrieve the pre-computed, cached result, eliminating redundant computation. This pattern is critical for managing the lifecycle of heavy objects, configuration data, or any resource where on-demand creation is preferred but repeated creation is wasteful.

## Lifecycle & Ownership
- **Creation:** As a static utility class, SupplierUtil is never instantiated. It is loaded into the JVM by the ClassLoader upon its first use.
- **Scope:** The class and its static methods are available for the entire application lifetime after being loaded.
- **Destruction:** The class is unloaded from memory when its ClassLoader is garbage collected, which typically occurs during application shutdown.

## Internal State & Concurrency
- **State:** SupplierUtil is entirely stateless. It contains no member fields and its methods are pure functions that operate exclusively on their input arguments.
- **Thread Safety:** The SupplierUtil class itself is unconditionally thread-safe. Calls to its static methods from multiple threads will not interfere with each other.

    **WARNING:** While the factory method is thread-safe, the thread safety of the returned CachedSupplier object is a separate concern. The CachedSupplier implementation is designed to be thread-safe, ensuring that the underlying delegate is invoked only once even under concurrent access. However, the delegate Supplier provided by the caller is not guaranteed to be thread-safe and should not rely on any specific thread context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| cache(Supplier<T> delegate) | CachedSupplier<T> | O(1) | Wraps a delegate Supplier to create a memoized, lazy-loading version. The returned supplier will only invoke the delegate's get method on the first call. |

## Integration Patterns

### Standard Usage
This utility should be used to defer and cache the result of an expensive operation, such as loading a resource from disk, performing a complex calculation, or initializing a major subsystem.

```java
// Define a supplier for a resource-intensive operation
Supplier<GameConfiguration> configLoader = () -> {
    System.out.println("Executing expensive configuration load...");
    return loadConfigurationFromDisk();
};

// Wrap the expensive operation in a cached supplier
Supplier<GameConfiguration> cachedConfig = SupplierUtil.cache(configLoader);

// First access: The expensive operation is executed
GameConfiguration c1 = cachedConfig.get(); // "Executing expensive configuration load..." is printed

// Second access: The cached result is returned instantly
GameConfiguration c2 = cachedConfig.get(); // Nothing is printed; the result is reused
```

### Anti-Patterns (Do NOT do this)
- **Wrapping Trivial Operations:** Avoid using SupplierUtil to cache operations that are already computationally inexpensive, such as returning a constant or a pre-existing object. The overhead of the caching wrapper and its synchronization logic may negate any potential performance gain.
- **Relying on Side-Effects:** The delegate Supplier's primary purpose should be to produce a value. Do not wrap a Supplier that relies on repeated side-effects (e.g., incrementing a metric on every call). The caching mechanism will prevent the side-effect from occurring on subsequent calls, leading to incorrect application behavior.

## Data Pipeline
SupplierUtil is a factory, not a data processor. The following describes the data flow for the CachedSupplier object it creates.

> **First Invocation:**
> Call to `CachedSupplier.get()` -> Internal lock acquired -> Delegate `Supplier.get()` is invoked -> Result is stored in a private field -> Internal lock released -> Result is returned

> **Subsequent Invocations:**
> Call to `CachedSupplier.get()` -> Read of private field -> Result is returned immediately


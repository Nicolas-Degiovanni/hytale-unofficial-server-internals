---
description: Architectural reference for ValidatorCache
---

# ValidatorCache<T>

**Package:** com.hypixel.hytale.codec.validation
**Type:** Stateful Helper

## Definition
```java
// Signature
public class ValidatorCache<T> {
```

## Architecture & Concepts
The ValidatorCache is a performance-optimization component within the data serialization and validation framework. Its primary role is to act as a factory and cache for specialized validator objects, preventing the performance overhead associated with repeated object instantiation during intensive operations like data decoding or network packet processing.

Architecturally, this class implements a lazy initialization pattern. It is constructed with a single, base Validator for a generic type T. It then provides methods to derive more complex validators for collections of T, such as arrays, arrays of arrays, and map keys or values. These derived validators are created only on their first request and are subsequently cached for the lifetime of the ValidatorCache instance.

This component is not intended for direct use by feature developers. Instead, it serves as a foundational building block for higher-level systems, such as a central CodecRegistry or ValidationManager, which manage the validation logic for numerous data types across the application. By centralizing the creation of collection-based validators, the system ensures efficiency and consistency.

### Lifecycle & Ownership
-   **Creation:** A ValidatorCache is instantiated by a managing service that oversees validation for a specific data type. For example, a system responsible for validating PlayerProfile objects would create and own a single ValidatorCache<PlayerProfile> instance.
-   **Scope:** The lifecycle of a ValidatorCache is tightly coupled to the base Validator it encapsulates. It is designed to be long-lived, persisting as long as validation for the associated data type is required.
-   **Destruction:** The object is subject to standard garbage collection. It holds no native resources and requires no explicit cleanup. It is destroyed when its owning service is destroyed and all references to it are released.

## Internal State & Concurrency
-   **State:** The ValidatorCache is a stateful, mutable object. Its internal fields for derived validators are initially null and are populated on-demand. Once a validator is created and cached, that part of the state becomes stable, but the object's overall state changes over its initial uses.

-   **Thread Safety:** **WARNING:** This class is **not thread-safe**. The internal lazy-initialization logic follows a check-then-act pattern (e.g., `if (this.field == null) { this.field = new ...; }`) which is vulnerable to race conditions. If multiple threads access a getter method concurrently for an uninitialized validator, it is possible for multiple validator instances to be created, leading to wasted resources and unpredictable behavior.

    All access to a shared ValidatorCache instance **must** be synchronized externally or confined to a single thread.

## API Surface
The public API consists exclusively of getter methods that provide access to the base validator and its derived, cached counterparts.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValidator() | Validator<T> | O(1) | Returns the original, base validator provided at construction. |
| getArrayValidator() | ArrayValidator<T> | Amortized O(1) | Lazily creates and returns a validator for an array of T. |
| getArrayOfArrayValidator() | ArrayValidator<T[]> | Amortized O(1) | Lazily creates and returns a validator for an array of arrays of T. |
| getMapKeyValidator() | MapKeyValidator<T> | Amortized O(1) | Lazily creates and returns a validator for map keys of type T. |
| getMapArrayKeyValidator() | MapKeyValidator<T[]> | Amortized O(1) | Lazily creates and returns a validator for map keys of type T[]. |
| getMapValueValidator() | MapValueValidator<T> | Amortized O(1) | Lazily creates and returns a validator for map values of type T. |
| getMapArrayValueValidator() | MapValueValidator<T[]> | Amortized O(1) | Lazily creates and returns a validator for map values of type T[]. |

## Integration Patterns

### Standard Usage
The ValidatorCache should be managed by a central service. The service retrieves the cache and then uses it to obtain the specific validator needed for a complex data structure.

```java
// Assume 'validationManager' holds caches for different types
ValidatorCache<PlayerProfile> profileCache = validationManager.getCache(PlayerProfile.class);

// Get a validator for a Map<String, PlayerProfile[]>
// Note: This example is conceptual; the key validator would come from a different cache.
MapValueValidator<PlayerProfile[]> valueValidator = profileCache.getMapArrayValueValidator();

// ... use the validator to process data
```

### Anti-Patterns (Do NOT do this)
-   **Per-Operation Instantiation:** Do not create a new ValidatorCache for each validation task. This completely negates the caching benefit and is less efficient than not using the cache at all.

    ```java
    // ANTI-PATTERN: Creates a new cache just to be thrown away.
    Validator<MyType> baseValidator = ...;
    Validator<MyType[]> arrayValidator = new ValidatorCache<>(baseValidator).getArrayValidator();
    ```

-   **Unsynchronized Multi-threaded Access:** Do not share a single ValidatorCache instance across threads without external locking. This will cause race conditions.

    ```java
    // ANTI-PATTERN: Two threads accessing the same cache can cause issues.
    ExecutorService executor = Executors.newFixedThreadPool(2);
    ValidatorCache<Data> sharedCache = getSharedCache();

    executor.submit(() -> sharedCache.getArrayValidator().validate(data1)); // Race condition
    executor.submit(() -> sharedCache.getArrayValidator().validate(data2)); // Race condition
    ```

## Data Pipeline
The ValidatorCache is not a direct participant in a data flow pipeline. Instead, it acts as a factory that provisions components (validators) used by pipeline stages.

> **Conceptual Flow:**
>
> Validation Service needs to validate `Map<String, T[]>` -> Requests `ValidatorCache<T>` -> **ValidatorCache.getMapArrayValueValidator()** -> Returns cached or new `MapValueValidator<T[]>` -> Validation Service uses validator to process data payload.


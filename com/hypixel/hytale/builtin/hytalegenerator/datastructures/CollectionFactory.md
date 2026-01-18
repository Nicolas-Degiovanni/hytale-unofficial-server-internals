---
description: Architectural reference for CollectionFactory
---

# CollectionFactory

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures
**Type:** Utility

## Definition
```java
// Signature
public class CollectionFactory {
```

## Architecture & Concepts
The CollectionFactory is a stateless utility class designed to provide convenient and robust factory methods for creating common Java Collection instances. Its primary role is to reduce boilerplate code and enforce non-null constraints at the point of creation, which improves code clarity and prevents downstream NullPointerExceptions.

This class is not part of a larger subsystem but serves as a foundational, low-level helper used across various engine components and game logic modules. It embodies a design principle of providing expressive and safe constructors for data structures, similar to patterns found in libraries like Guava or the Kotlin Standard Library.

## Lifecycle & Ownership
- **Creation:** As a class containing only static methods, CollectionFactory is never instantiated. The Java Virtual Machine's ClassLoader loads it into memory upon its first use.
- **Scope:** The class and its methods are available for the entire application lifecycle once loaded.
- **Destruction:** The class is unloaded from memory when its ClassLoader is garbage collected, which typically coincides with application shutdown.

## Internal State & Concurrency
- **State:** CollectionFactory is entirely stateless. It contains no member variables and all operations are performed on method-local variables and input arguments.
- **Thread Safety:** This class is inherently thread-safe. Calls to its static methods from multiple threads will not interfere with each other. Each invocation of a factory method creates a new, distinct collection instance that is not shared. The thread safety of the *returned collection* is the responsibility of the calling code, as is standard for the Java Collections Framework.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hashSetOf(T... elements) | Set<T> | O(N) | Creates a new HashSet instance from a variable number of elements. Throws NullPointerException if any of the provided elements are null. |

## Integration Patterns

### Standard Usage
The intended use is to call the static methods directly to create pre-populated collections in a single, expressive line.

```java
// How a developer should normally use this
import com.hypixel.hytale.builtin.hytalegenerator.datastructures.CollectionFactory;

Set<String> allowedBiomes = CollectionFactory.hashSetOf("Forest", "Plains", "Mountains");

if (allowedBiomes.contains(currentBiome)) {
    // ... logic
}
```

### Anti-Patterns (Do NOT do this)
- **Passing Null Elements:** The factory methods are designed to be null-hostile to ensure the integrity of the created collections. Passing a null value will result in an immediate runtime exception. This is by design.
- **Attempted Instantiation:** This class is not designed to be instantiated. While the provided code does not include a private constructor to enforce this, attempting to create an instance (e.g., `new CollectionFactory()`) serves no purpose and violates its design as a static utility.

## Data Pipeline
This component acts as a source node in a data flow, not a processing stage. It manufactures data structures that are then used by other systems.

> Flow:
> Varargs Array of Elements -> **CollectionFactory.hashSetOf** -> New HashSet Instance -> Application Logic


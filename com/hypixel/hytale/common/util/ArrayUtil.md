---
description: Architectural reference for ArrayUtil
---

# ArrayUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class ArrayUtil {
```

## Architecture & Concepts

ArrayUtil is a foundational, stateless utility class designed to provide a centralized and performant set of operations for native Java arrays. It serves as a critical component in the engine's low-level data manipulation toolkit, augmenting the standard `java.util.Arrays` with Hytale-specific logic and common boilerplate-reducing patterns.

The primary architectural driver for this class is performance. In many engine systems, such as entity management, rendering, and networking, the overhead of using Java Collections like `ArrayList` (which involves boxing primitives and pointer indirection) is unacceptable. ArrayUtil provides direct, efficient manipulation of contiguous memory blocks represented by arrays.

Key architectural features include:
*   **Immutability-Focused Operations:** Methods like `append`, `remove`, and `combine` do not modify the input arrays. Instead, they return new, correctly-sized arrays. This functional approach promotes predictable state management and reduces side effects, though it carries performance implications that developers must manage.
*   **In-Place Operations:** For performance-critical scenarios where allocation is undesirable, methods like `shuffleArray` are provided to mutate the array's contents directly.
*   **Memory Optimization:** The class pre-allocates and exposes a set of `EMPTY_*_ARRAY` constants. This is a deliberate memory optimization to prevent the repeated allocation of zero-length arrays throughout the codebase, reducing garbage collector pressure. Systems should return these static instances instead of creating new empty arrays.

## Lifecycle & Ownership

As a static utility class, ArrayUtil does not have a traditional object lifecycle. It is not designed to be instantiated.

*   **Creation:** The ArrayUtil class is loaded into the JVM by the system's ClassLoader when it is first referenced by any part of the engine. Its static final fields, such as `EMPTY_STRING_ARRAY`, are initialized at this time.
*   **Scope:** Application-wide. Once loaded, its static methods and constants are available for the entire duration of the application's runtime.
*   **Destruction:** The class and its static members are unloaded from the JVM when the application terminates.

## Internal State & Concurrency

*   **State:** ArrayUtil is completely **stateless**. It contains no mutable static or instance fields. All operations are pure functions whose output depends solely on their input arguments.
*   **Thread Safety:** This class is inherently **thread-safe**. Because it is stateless, its methods can be invoked concurrently from any number of threads without risk of data corruption or race conditions within ArrayUtil itself.

**WARNING:** While the methods themselves are thread-safe, the arrays passed to them are not. The caller is responsible for ensuring that no other thread is modifying an array that is being read or written to by a method in this class. Failure to provide proper synchronization on the calling side will lead to unpredictable behavior.

## API Surface

The API provides a mix of allocation-heavy convenience methods and in-place performance-oriented utilities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| append(arr, t) | T[] | O(N) | Creates a **new** array containing all elements of the original plus the new element. Incurs allocation and copy cost. |
| remove(arr, index) | T[] | O(N) | Creates a **new** array with the element at the specified index removed. Incurs allocation and copy cost. |
| combine(a1, a2) | T[] | O(N+M) | Creates a **new** array by concatenating two source arrays. |
| shuffleArray(ar, ...) | void | O(N) | Shuffles the elements of the provided array **in-place** using the Fisher-Yates algorithm. Does not allocate a new array. |
| grow(oldSize) | int | O(1) | Calculates a new, larger capacity for an array, typically 1.5x the old size. Used for manual buffer growth logic. |
| copyAndMutate(...) | EndType[] | O(N) | A functional-style transform that creates a new array by applying an adapter function to each element of a source array. |

## Integration Patterns

### Standard Usage

ArrayUtil should be used for infrequent or bulk modifications of arrays where the clarity and safety of receiving a new array instance is desired.

```java
// Example: Adding a new component to an entity's component array
Component[] currentComponents = entity.getComponents();
Component newComponent = new PositionComponent();

// The append method returns a new array, which replaces the old one
Component[] updatedComponents = ArrayUtil.append(currentComponents, newComponent);
entity.setComponents(updatedComponents);
```

### Anti-Patterns (Do NOT do this)

*   **Modification in Loops:** The most critical anti-pattern is using allocation-based methods like `append` or `remove` inside a tight loop. Each call creates a new array and copies all existing elements, resulting in O(N^2) complexity and immense pressure on the garbage collector.

    ```java
    // CRITICAL ANTI-PATTERN: Do not do this.
    String[] data = ArrayUtil.EMPTY_STRING_ARRAY;
    for (int i = 0; i < 10000; i++) {
        // This allocates 10,000 arrays and performs millions of element copies.
        data = ArrayUtil.append(data, "value" + i);
    }
    ```
    For this use case, a standard `ArrayList` or a pre-allocated array is the correct architectural choice.

*   **Direct Instantiation:** Do not create an instance of this class. All methods are static. `new ArrayUtil()` serves no purpose and is a code smell.

## Data Pipeline

ArrayUtil does not represent a stage in a data pipeline. Rather, it is a low-level toolkit used by various components *within* a pipeline to manage the data they are processing. It provides the fundamental mechanics for resizing or altering the data containers that flow between major systems.

> **Conceptual Flow:**
>
> Network Packet -> Deserializer -> **ArrayUtil.combine(existing, new)** -> Game State Update -> Renderer


---
description: Architectural reference for Tri
---

# Tri

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class Tri<A, B, C> {
```

## Architecture & Concepts
The Tri class is a generic, immutable container designed to hold a fixed-size group of three potentially heterogeneous objects. It functions as a type-safe tuple, providing a lightweight mechanism for bundling related data without the overhead of defining a dedicated class for every ad-hoc combination.

Architecturally, Tri serves as a fundamental data structure, primarily used as a Data Transfer Object (DTO) between different subsystems, especially within the builder tools and prefab editor context. Its primary design goal is to enforce immutability, ensuring that data passed between components cannot be unexpectedly modified, which simplifies state management and enhances thread safety. It is a more robust and explicit alternative to using a raw `Object[]` or a `List` where the size and types of the elements are fixed and known at compile time.

## Lifecycle & Ownership
- **Creation:** Instances are created directly by client code using the `new` keyword. There is no factory or service locator responsible for its instantiation. It is a transient object, created on-demand.
- **Scope:** The lifecycle of a Tri instance is bound to the scope of the object that created it. It is typically short-lived, existing only for the duration of a single method call or operation. It is not managed by any container and does not persist across sessions.
- **Destruction:** Ownership is relinquished when all references to the instance are lost. The object is then eligible for garbage collection by the JVM. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. The internal fields `left`, `middle`, and `right` are declared as `final` and are assigned only once during construction. The class provides no methods to modify its internal state after instantiation. This design guarantees that an instance of Tri is a constant, predictable value object.

- **Thread Safety:** **Conditionally Thread-Safe**. The Tri container itself is inherently thread-safe due to its immutability. An instance can be safely shared and read by multiple threads without external synchronization.
    - **WARNING:** The thread safety of the *contained objects* (types A, B, and C) is not guaranteed by Tri. If a mutable object is stored within a Tri instance that is shared across threads, the client code is responsible for ensuring the thread safety of that mutable object.

## API Surface
The public contract is minimal, consisting of the constructor and three accessor methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Tri(A, B, C) | constructor | O(1) | Constructs a new immutable Tri instance. |
| getLeft() | A | O(1) | Returns the first element of the triplet. |
| getMiddle() | B | O(1) | Returns the second element of the triplet. |
| getRight() | C | O(1) | Returns the third element of the triplet. |

## Integration Patterns

### Standard Usage
Tri is intended for grouping related data for method arguments or return values, particularly when creating a dedicated class would be excessive.

```java
// Example: Returning a block's position, type, and orientation from a query.
public Tri<Vector3i, BlockType, Direction> getBlockInfoAt(int x, int y, int z) {
    Vector3i position = new Vector3i(x, y, z);
    BlockType type = world.getBlockType(position);
    Direction orientation = world.getBlockOrientation(position);

    // Bundle the results into a single, immutable object for safe transport.
    return new Tri<>(position, type, orientation);
}

// Usage:
Tri<Vector3i, BlockType, Direction> blockData = getBlockInfoAt(10, 20, 30);
System.out.println("Block at " + blockData.getLeft() + " is " + blockData.getMiddle());
```

### Anti-Patterns (Do NOT do this)
- **Replacing Well-Defined Models:** Do not use Tri as a substitute for a proper domain object. If the three elements have a strong, permanent semantic relationship (e.g., PlayerHealth, PlayerStamina, PlayerMana), a dedicated `PlayerStats` class is vastly more readable and maintainable.
- **Storing Mutable Payloads in Concurrent Systems:** Avoid storing mutable objects in a Tri instance that will be shared across threads unless those objects are themselves thread-safe. This violates the principle of effective immutability and can lead to severe concurrency bugs.
- **Deep Nesting:** Structures like `Tri<A, Tri<B, C, D>, E>` should be avoided. They are difficult to read, debug, and maintain. If your data structure is this complex, it requires a dedicated class definition.

## Data Pipeline
As a value object, Tri does not process data itself. Instead, it acts as a data packet or payload that is passed through a pipeline.

> Flow:
> World Query System -> **Tri (as data payload)** -> Game Logic -> Rendering Engine


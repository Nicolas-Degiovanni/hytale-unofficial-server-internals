---
description: Architectural reference for NullSpace
---

# NullSpace

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.voxelspace
**Type:** Singleton / Utility

## Definition
```java
// Signature
public class NullSpace<V> implements VoxelSpace<V> {
```

## Architecture & Concepts
The NullSpace class is a specific implementation of the VoxelSpace interface that embodies the **Null Object** design pattern. Its primary architectural purpose is to represent a non-existent or empty volume of voxels, providing a safe, inert alternative to returning a `null` reference.

By offering a concrete object with no-op behavior, NullSpace allows client systems to interact with a VoxelSpace reference without the need for explicit null checks. This simplifies consumer logic, reduces boilerplate code, and critically, prevents NullPointerExceptions that would arise from attempting to operate on a null VoxelSpace. It is the canonical representation of "nothing" within the voxel data structure system.

All mutation methods return `false` or do nothing, all query methods return default empty values (0, null, false), and iterators will complete instantly without performing any actions.

### Lifecycle & Ownership
- **Creation:** The single NullSpace instance is created statically during class loading by the JVM. It is not instantiated by any application-level factory or dependency injection container. Its lifecycle is tied directly to the JVM process itself.
- **Scope:** Application-wide. A single, shared instance is returned by the static `instance()` factory method, persisting for the entire runtime of the application.
- **Destruction:** The instance is garbage collected when the application's class loader is unloaded, typically during JVM shutdown. There is no manual destruction or cleanup mechanism.

## Internal State & Concurrency
- **State:** NullSpace is completely **stateless and immutable**. It contains no fields that store data related to a voxel volume. Its behavior is constant and predictable.
- **Thread Safety:** This class is inherently **thread-safe**. As a stateless singleton, it can be safely shared and accessed by multiple threads concurrently without any need for external synchronization or internal locking.

## API Surface
The API surface strictly adheres to the VoxelSpace interface, providing no-op or default-value implementations for every method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| instance() | NullSpace | O(1) | Returns the global, static instance of NullSpace. |
| set(content, x, y, z) | boolean | O(1) | A no-op operation. Always returns `false` to indicate the set operation failed. |
| getContent(x, y, z) | V | O(1) | Always returns `null`, as there is no content at any coordinate. |
| isInsideSpace(x, y, z) | boolean | O(1) | Always returns `false`, as no coordinate is considered inside this zero-volume space. |
| forEach(action) | void | O(1) | A no-op operation. The provided consumer is never invoked. |
| sizeX() | int | O(1) | Always returns 0. |

## Integration Patterns

### Standard Usage
NullSpace should be used as a return value from functions that might otherwise return a `null` VoxelSpace. This allows the calling code to operate on the result uniformly without conditional checks.

```java
// A world generation function that returns an empty space for invalid requests
public VoxelSpace<Block> getStructureAt(Vector3i position) {
    if (!isValidStructureLocation(position)) {
        // Return the safe, inert NullSpace instead of null
        return NullSpace.instance();
    }
    // ... logic to return a real VoxelSpace
}

// Consumer code is simplified and safe from NullPointerExceptions
VoxelSpace<Block> structure = getStructureAt(somePosition);
structure.forEach(block -> {
    // This code will execute for a real structure but will
    // simply be skipped for a NullSpace, without a crash.
    world.setBlock(block.position, block.type);
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create an instance using reflection. Always use the static `NullSpace.instance()` method.
- **Type Checking:** Avoid writing code that explicitly checks for this type. The purpose of the Null Object pattern is to eliminate such checks.

   ```java
   // ANTI-PATTERN: This defeats the purpose of the class.
   if (space instanceof NullSpace) {
       // Do nothing
   } else {
       space.forEach(...);
   }

   // CORRECT: Let polymorphism handle the behavior.
   space.forEach(...);
   ```

## Data Pipeline
NullSpace acts as a terminal node or a "sink" in any data pipeline. It accepts method calls but does not store, process, or forward any data. It effectively and safely terminates any operation sequence that receives it.

> Flow:
> World Generator -> Request for Optional Voxel Data -> **NullSpace** (Returned for non-existent data) -> Voxel Processor (Executes no-op methods, terminating the chain)


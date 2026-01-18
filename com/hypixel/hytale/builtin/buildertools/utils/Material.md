---
description: Architectural reference for Material
---

# Material

**Package:** com.hypixel.hytale.builtin.buildertools.utils
**Type:** Utility / Value Object

## Definition
```java
// Signature
public final class Material {
```

## Architecture & Concepts

The Material class is an immutable value object that serves as a high-level, unified representation of a single point in the game world. It is a foundational component of the Builder Tools system, abstracting the engine's underlying integer-based identifiers for blocks and fluids into a more robust, type-safe container.

Its core architectural purpose is to decouple world-editing logic from the raw data representation of world state. Instead of passing around separate integers for block IDs, fluid IDs, and rotation data, systems can operate on a single, self-contained Material instance. This simplifies APIs and reduces the potential for errors, such as attempting to apply a rotation to a fluid.

The class design enforces a strict separation of concerns: a Material can be a block (with an optional rotation), a fluid (with a fluid level), or empty, but never a combination. This is managed internally through mutually exclusive state fields.

Creation is exclusively controlled by static factory methods, which act as the sole entry points for translating various data sources—such as string identifiers or procedural patterns—into a valid Material instance. This pattern centralizes parsing, asset lookup, and validation logic.

## Lifecycle & Ownership

-   **Creation:** Instances are never created directly via a constructor. They are exclusively instantiated through static factory methods like `block`, `fluid`, `fromKey`, or `fromPattern`. These methods are typically invoked by higher-level systems like command parsers or procedural generation tools. The special `EMPTY` instance is a static singleton, created at class-load time.
-   **Scope:** Material objects are transient and short-lived. They are designed to be created on-demand for a specific operation (e.g., a single world edit command) and are not intended to be cached or held for long periods. Once the operation is complete, they are eligible for garbage collection.
-   **Destruction:** Cleanup is managed entirely by the Java Garbage Collector. As immutable objects holding only primitive data, they do not manage any external resources that would require manual destruction.

## Internal State & Concurrency

-   **State:** The Material class is **strictly immutable**. All internal fields (`blockId`, `fluidId`, `fluidLevel`, `rotation`) are final and are set only once within the private constructor. An instance's state cannot be changed after creation.
-   **Thread Safety:** As a direct result of its immutability, Material is **inherently thread-safe**. Instances can be safely passed between and read by multiple threads without any external synchronization or locking mechanisms. This makes it suitable for use in parallelized world generation or modification tasks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| block(int blockId, int rotation) | Material | O(1) | Factory method to create a block material. |
| fluid(int fluidId, byte fluidLevel) | Material | O(1) | Factory method to create a fluid material. |
| fromKey(String key) | Material | O(1) avg | Parses a string key (e.g., "hytale:stone") into a Material. Involves asset map lookups. Returns null on failure. |
| fromPattern(BlockPattern pattern, Random random) | Material | O(1) avg | Resolves a BlockPattern into a single Material instance. Returns null on failure. |
| isBlock() | boolean | O(1) | Returns true if the material represents a block and not a fluid. |
| isFluid() | boolean | O(1) | Returns true if the material represents a fluid. |
| isEmpty() | boolean | O(1) | Returns true if the material represents empty space. |
| getBlockId() | int | O(1) | Retrieves the raw block ID. Returns 0 if not a block. |
| getRotation() | int | O(1) | Retrieves the raw rotation value. Returns 0 if not a block or has no rotation. |

## Integration Patterns

### Standard Usage

The primary integration pattern is to use the static factory methods to create a Material from a known source, inspect its type, and then apply its properties to the game world.

```java
// Standard usage for a command that sets a block
Material material = Material.fromKey("hytale:stone[axis=y]");

if (material == null) {
    // Handle invalid input key
    player.sendMessage("Error: Unknown material.");
    return;
}

if (material.isBlock()) {
    // Correctly identified as a block, proceed with world edit
    world.setBlock(targetPosition, material.getBlockId(), material.getRotation());
} else if (material.isFluid()) {
    // Handle cases where a fluid was specified when a block was expected
    player.sendMessage("Error: Expected a block, but got a fluid.");
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The constructor is private for a reason. Attempting to instantiate with `new Material()` is a compile-time error and violates the class's design contract. Always use the provided static factory methods.
-   **State Assumption:** Never assume a Material instance is of a certain type. Always use the `isBlock()`, `isFluid()`, and `isEmpty()` guards before attempting to use its data in a type-specific context. Accessing `getBlockId()` on a fluid material will not crash, but it will return 0, leading to incorrect behavior if not handled properly.
-   **Ignoring Nulls:** The `fromKey` and `fromPattern` methods can return null if the provided input is invalid or cannot be resolved. Failure to perform a null check will result in a NullPointerException.

## Data Pipeline

The Material class acts as a translation and validation step within a larger data flow, typically initiated by a user or system action.

> Flow:
> User Input (e.g., Command `//set hytale:water`) -> Command Parser -> **Material.fromKey("hytale:water")** -> World Edit System -> Engine World State Update


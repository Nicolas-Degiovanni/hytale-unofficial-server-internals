---
description: Architectural reference for VariantRotation
---

# VariantRotation

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Enum / Strategy Pattern

## Definition
```java
// Signature
public enum VariantRotation implements NetworkSerializable<com.hypixel.hytale.protocol.VariantRotation> {
```

## Architecture & Concepts
The VariantRotation enum is a foundational component of the block asset system, implementing the **Strategy pattern** to define how different categories of blocks can be oriented within the game world. Each enum constant, such as Wall, Pipe, or NESW, encapsulates a complete and distinct set of rotation rules, constraints, and transformation logic.

Architecturally, this enum decouples the generic block placement and manipulation logic from the specific rotational behaviors of individual blocks. Instead of a monolithic service containing complex conditional logic for every block type, a block's asset definition simply references a VariantRotation constant. The world interaction systems then delegate the task of calculating valid orientations to the strategy object provided by this enum.

This component acts as a translation layer between a player's intended orientation (derived from their camera angle) and the discrete, valid set of rotations a specific block model supports. For example, the Wall strategy will automatically snap any player rotation to one of the four cardinal directions, while the UpDown strategy will only permit orientations facing up or down.

## Lifecycle & Ownership
- **Creation:** As a Java enum, all instances (None, Wall, Pipe, etc.) are constructed and initialized by the JVM during class loading. They are compile-time constants, not dynamically created objects.
- **Scope:** The instances are static, global, and persist for the entire lifetime of the server application. They are effectively permanent singletons managed by the JVM.
- **Destruction:** The instances are destroyed only when the JVM shuts down. There is no manual memory management or cleanup required.

## Internal State & Concurrency
- **State:** Each VariantRotation constant is **deeply immutable**. Its internal state, including the array of valid RotationTuples and the functional interfaces defining its logic (verify, rotateX, rotateZ), is assigned during static initialization and never changes.
- **Thread Safety:** This class is inherently **thread-safe**. All of its methods are pure functions that operate solely on their inputs to produce an output. They do not modify any internal state, making them safe to be called concurrently from any thread without locks or other synchronization primitives.

## API Surface
The public API provides a contract for querying valid rotations and applying rotational transformations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRotations() | RotationTuple[] | O(1) | Returns the canonical array of valid rotations for this strategy. **Warning:** The returned array is not a copy and must not be modified. |
| rotateX(pair, rotation) | RotationTuple | O(1) | Applies a rotation around the X-axis according to the strategy's rules. |
| rotateZ(pair, rotation) | RotationTuple | O(1) | Applies a rotation around the Z-axis according to the strategy's rules. |
| verify(pair) | RotationTuple | O(1) | Takes an arbitrary rotation and snaps it to the nearest valid rotation defined by the strategy. This is critical for normalizing player input. |
| toPacket() | com.hypixel.hytale.protocol.VariantRotation | O(1) | Serializes the enum constant to its corresponding network protocol representation. |

## Integration Patterns

### Standard Usage
The primary integration point is within the server's block placement and interaction logic. When a block is placed, its asset definition is queried for its VariantRotation strategy, which is then used to calculate the final, valid orientation.

```java
// PSEUDOCODE: Demonstrates the intended use within a world service
BlockType blockToPlace = assetManager.getBlock("hytale:oak_log");
VariantRotation rotationStrategy = blockToPlace.getRotationStrategy(); // e.g., VariantRotation.Pipe

// Get the player's intended orientation
RotationTuple desiredRotation = player.getLookDirection().toRotationTuple();

// Use the strategy to sanitize and finalize the rotation
RotationTuple finalRotation = rotationStrategy.verify(desiredRotation);

// Place the block in the world with the corrected orientation
world.setBlock(targetPosition, blockToPlace, finalRotation);
```

### Anti-Patterns (Do NOT do this)
- **External Type Checking:** Do not use `switch` statements or `instanceof` checks on a VariantRotation instance. The purpose of this design is to rely on the polymorphic behavior of its methods. Let the enum constant itself execute the correct logic.
    ```java
    // BAD: Violates the Strategy pattern
    RotationTuple finalRotation;
    switch (rotationStrategy) {
        case Pipe:
            finalRotation = handlePipeRotation(desiredRotation);
            break;
        case Wall:
            finalRotation = handleWallRotation(desiredRotation);
            break;
    }

    // GOOD: Polymorphic and extensible
    RotationTuple finalRotation = rotationStrategy.verify(desiredRotation);
    ```
- **State Modification:** Never attempt to modify the contents of the array returned by `getRotations`. It is a shared, static resource, and modifying it will lead to undefined behavior across the entire server.

## Data Pipeline
VariantRotation functions as a transformation and validation step within the broader block interaction data pipeline.

> Flow:
> Player Input (Place/Rotate Action) -> World Interaction Service -> Block Asset Lookup -> **VariantRotation Strategy** -> `verify()` Method -> Validated RotationTuple -> World State Update -> Network Packet to Clients


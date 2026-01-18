---
description: Architectural reference for RandomDirectionality
---

# RandomDirectionality

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.directionality
**Type:** Transient

## Definition
```java
// Signature
public class RandomDirectionality extends Directionality {
```

## Architecture & Concepts
The RandomDirectionality class is a strategy component within the procedural world generation framework. It implements the abstract concept of Directionality, providing a specific behavior for determining the orientation of a Pattern placed in the world.

Its core responsibility is to assign a random, yet deterministic, rotation to a Pattern at any given world coordinate. This is fundamental for creating varied and organic-looking environments. By using a coordinate-based seed, it guarantees that regenerating the same world with the same master seed will produce the exact same rotations for every object, ensuring world consistency.

This class decouples the structural definition of a Pattern from its orientation logic. A single Pattern can be associated with RandomDirectionality to make it appear randomly rotated throughout the world, or with a different Directionality implementation for other behaviors (e.g., always facing north).

## Lifecycle & Ownership
-   **Creation:** Instantiated by higher-level world generation controllers or configuration objects when defining the placement rules for a specific Pattern. It is not a globally managed service.
-   **Scope:** The object's lifetime is tied to the generator or rule set that created it. It typically persists for the duration of a specific world generation phase (e.g., placing all instances of a certain type of tree).
-   **Destruction:** Eligible for garbage collection as soon as the parent generator completes its task and releases its reference.

## Internal State & Concurrency
-   **State:** The internal state is established at construction and is **immutable**. The list of possible rotations is explicitly wrapped in an unmodifiable list, and the associated Pattern and SeedGenerator are final.
-   **Thread Safety:** This class is inherently **thread-safe**. Its immutable nature prevents data races. The primary method, getRotationAt, is re-entrant and creates a new FastRandom instance for each invocation, avoiding shared mutable state. It can be safely used across multiple world generation threads operating on different chunks simultaneously.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGeneralPattern() | Pattern | O(1) | Returns the Pattern this directionality is applied to. |
| getReadRangeWith(Scanner) | Vector3i | O(1) | Delegates to the scanner to determine the spatial range required. |
| getPossibleRotations() | List<PrefabRotation> | O(1) | Returns the static, unmodifiable list of four cardinal rotations. |
| getRotationAt(Pattern.Context) | PrefabRotation | O(1) | Calculates and returns a deterministic, random rotation for the given context. |

## Integration Patterns

### Standard Usage
This class is intended to be instantiated once per Pattern that requires random orientation and then used repeatedly by the generation system.

```java
// A generator configures a tree pattern with random directionality
Pattern treePattern = getSomeTreePattern();
Directionality randomTrees = new RandomDirectionality(treePattern, worldSeed);

// Later, during chunk population...
Pattern.Context placementContext = new Pattern.Context(position, ...);
PrefabRotation rotation = randomTrees.getRotationAt(placementContext);
world.place(treePattern, position, rotation);
```

### Anti-Patterns (Do NOT do this)
-   **Per-Call Instantiation:** Do not create a new RandomDirectionality for every block or position. This is inefficient and defeats the purpose of encapsulating the configuration.
-   **Ignoring World Seed:** Providing a non-deterministic seed (e.g., from system time) will break world reproducibility. The seed should be derived from the master world seed.

## Data Pipeline
RandomDirectionality acts as a deterministic function in the world generation pipeline, transforming a position into an orientation.

> Flow:
> World Generator identifies a placement location -> Creates a Pattern.Context with the position -> **RandomDirectionality.getRotationAt(context)** -> Returns a PrefabRotation -> World Generator uses the rotation to place the final Pattern into the world data.


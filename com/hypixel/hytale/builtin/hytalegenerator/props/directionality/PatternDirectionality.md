---
description: Architectural reference for PatternDirectionality
---

# PatternDirectionality

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.directionality
**Type:** Transient

## Definition
```java
// Signature
public class PatternDirectionality extends Directionality {
```

## Architecture & Concepts
PatternDirectionality is a strategy object used within the procedural world generation framework to determine the orientation of an object based on its surrounding environment. It encapsulates the logic that maps four distinct environmental conditions—represented by Pattern objects—to the four cardinal rotations: North, South, East, and West.

This class acts as a sophisticated, context-aware decision-making component. Instead of a prop having a fixed rotation, a generator can use an instance of PatternDirectionality to ask, "Given the state of the world at this specific coordinate, what is the most logical rotation for this object?"

The core concept is to define what the world should look like for each of the four directions relative to the object. For example, for a door prop, the *southPattern* might check for an empty block to the south, while the other patterns check for solid walls. The class then evaluates these patterns at a given location and returns a valid rotation. The inclusion of a `startingDirection` in the constructor provides a frame of reference, allowing designers to define patterns relative to a prefab's "front" rather than absolute world directions.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor (`new PatternDirectionality(...)`). It is typically created during the initialization phase of a higher-level world generation component, such as a Prop or a specific generation pass. It is not managed by a service locator or dependency injection container.
- **Scope:** The object's lifetime is bound to its owner, which is almost always a configuration object for a piece of generated content. It is effectively an immutable configuration snapshot that persists as long as its parent generator is active.
- **Destruction:** The object is eligible for garbage collection once its owning configuration is no longer referenced. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state is established entirely within the constructor and is **immutable** thereafter. All fields are final, and the list of rotations is explicitly wrapped in an unmodifiable list. This design ensures that a PatternDirectionality instance is a stable, reusable configuration object.
- **Thread Safety:** This class is **conditionally thread-safe**. Because its internal state is immutable, instances can be safely shared and read by multiple generator threads. The primary method, getRotationAt, is safe for concurrent execution as it creates a new, deterministically-seeded FastRandom instance for each call, avoiding shared mutable state. Thread safety is contingent on the external Pattern.Context and the underlying world data source being safe for concurrent reads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGeneralPattern() | Pattern | O(1) | Returns a composite OrPattern combining all four directional patterns. Used for broad-phase culling. |
| getReadRangeWith(Scanner) | Vector3i | O(N) | Calculates the bounding box required to evaluate the general pattern. Complexity depends on the scanner and pattern. |
| getPossibleRotations() | List<PrefabRotation> | O(1) | Returns the static list of the four cardinal rotations configured at creation time. |
| getRotationAt(Pattern.Context) | PrefabRotation | O(M) | The primary evaluation method. Tests all four directional patterns against the world context and returns a single, randomly selected valid rotation, or null if no patterns match. Complexity M depends on the cost of matching the sub-patterns. |

## Integration Patterns

### Standard Usage
PatternDirectionality is intended to be instantiated once as part of a larger configuration, such as a Prop definition, and then reused for many placement checks.

```java
// In a Prop configuration or generator setup
Pattern south = new BlockPattern(Blocks.AIR);
Pattern north = new BlockPattern(Blocks.STONE);
Pattern east = new BlockPattern(Blocks.STONE);
Pattern west = new BlockPattern(Blocks.STONE);
int worldSeed = 12345;

// Create the strategy object once
Directionality directionality = new PatternDirectionality(
    OrthogonalDirection.S, south, north, east, west, worldSeed
);

// In the generation loop, for a given position...
Pattern.Context context = createSomehow(world, position);
PrefabRotation chosenRotation = directionality.getRotationAt(context);

if (chosenRotation != null) {
    // Place the prefab with the determined rotation
    placePrefab(position, chosenRotation);
}
```

### Anti-Patterns (Do NOT do this)
- **Instantiation-Per-Check:** Do not create a `new PatternDirectionality` inside a tight loop for every block being checked. This is highly inefficient. Create it once and reuse it.
- **Overly Complex Patterns:** The performance of the entire generator can be degraded by using excessively large or computationally expensive Patterns for the directional checks. The `getRotationAt` method is often in a hot path, and its performance is directly tied to the cost of its constituent `pattern.matches()` calls.
- **Ignoring Null Return:** The `getRotationAt` method can return null if no environmental conditions are met. Code must be robust and handle this case, as it signifies that the object cannot be placed at the target location with a valid orientation.

## Data Pipeline
The class functions as a transformation step within a larger world generation data flow, converting positional context into a rotational output.

> Flow:
> Generator identifies candidate `Vector3i` -> A `Pattern.Context` is created for that position -> Context is passed to **PatternDirectionality** -> The four internal patterns are matched against world data -> A valid `PrefabRotation` is selected deterministically -> The Generator receives the rotation for final placement.


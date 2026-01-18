---
description: Architectural reference for StaticDirectionality
---

# StaticDirectionality

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.directionality
**Type:** Immutable Value Object

## Definition
```java
// Signature
public class StaticDirectionality extends Directionality {
```

## Architecture & Concepts
StaticDirectionality is a concrete implementation of the **Directionality** strategy pattern. Its core architectural purpose is to enforce a fixed, predetermined orientation for a world generation component, represented by a **Pattern**.

Within the Hytale Generator framework, various Directionality strategies exist to allow procedural elements to adapt to their environment. StaticDirectionality represents the simplest case: an object that *never* changes its rotation. It acts as a configuration constant, ensuring that specific prefabs or patterns are placed with a consistent, predictable orientation.

This class is fundamental for placing canonical structures, such as landmarks, specific dungeon entrances, or any feature that must maintain a fixed facing (e.g., always facing north) regardless of the surrounding generation context. It decouples the concept of a pattern's shape from its orientation, providing a rigid and non-negotiable rotation rule to the generator's placement algorithms.

### Lifecycle & Ownership
-   **Creation:** Instantiated during the configuration phase of the world generator. It is typically created once when a specific procedural generation rule or "prop" is defined, not dynamically during the world-building process.
-   **Scope:** The object's lifetime is tied to the generator rule set it belongs to. As an immutable value object, it is passed by reference throughout the generation pipeline but its state never changes. It persists as long as its parent configuration is loaded.
-   **Destruction:** Eligible for garbage collection when the server unloads the world or the associated generator configuration is no longer in use.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields (rotation, pattern, possibleRotations) are declared final and are assigned exclusively during construction. The list of rotations is further protected by being wrapped in an unmodifiable collection, making the object's state deeply constant.

-   **Thread Safety:** **Fully thread-safe**. Its immutability guarantees that instances of StaticDirectionality can be safely shared and read by multiple, concurrent world-generation threads without locks or any other synchronization primitives. This is a critical design attribute for achieving high performance in a parallelized generator.

## API Surface
The public contract is minimal and designed for high-frequency access during world generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRotationAt(Context context) | PrefabRotation | O(1) | Returns the pre-configured rotation, ignoring the context parameter entirely. |
| getGeneralPattern() | Pattern | O(1) | Returns the Pattern associated with this fixed orientation. |
| getReadRangeWith(Scanner scanner) | Vector3i | O(N) | Delegates to the provided Scanner to calculate the bounding box for the Pattern. Complexity is dependent on the Scanner's implementation. |
| getPossibleRotations() | List<PrefabRotation> | O(1) | Returns an unmodifiable list containing the single, static rotation. |

## Integration Patterns

### Standard Usage
StaticDirectionality is used as a configuration component when defining a generator prop that must not rotate. It is constructed once and injected into the prop's definition.

```java
// In a generator configuration file or bootstrap sequence:

// 1. Define the pattern for a structure that must always face north.
Pattern northFacingAltar = PatternRegistry.get("ancient_altar");
PrefabRotation fixedRotation = PrefabRotation.NORTH;

// 2. Create the static directionality rule.
Directionality staticRule = new StaticDirectionality(fixedRotation, northFacingAltar);

// 3. Assign this rule to a higher-level generator object.
// The PropPlacer will now always use PrefabRotation.NORTH for this pattern.
prop.setDirectionality(staticRule);
```

### Anti-Patterns (Do NOT do this)
-   **Misuse for Dynamic Elements:** Do not use StaticDirectionality for features that are intended to adapt to terrain, connect to paths, or face other structures. Using it in such scenarios will lead to unnatural, disjointed world generation. For such cases, a more complex Directionality implementation must be used.
-   **Redundant Instantiation:** Avoid creating new instances of StaticDirectionality within a generation loop. As an immutable value object, it should be defined once during the initialization of a rule set and reused.

## Data Pipeline
StaticDirectionality does not process a stream of data. Instead, it acts as a static data provider *within* the larger world generation pipeline. It injects a constant value (the rotation) into the placement algorithm.

> Flow:
> Prop Placement Request -> **StaticDirectionality**.getRotationAt() -> Placement Algorithm -> Voxel Write Operation


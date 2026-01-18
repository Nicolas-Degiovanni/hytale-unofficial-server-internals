---
description: Architectural reference for Directionality
---

# Directionality

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.directionality
**Type:** Utility / Abstract Base Class

## Definition
```java
// Signature
public abstract class Directionality {
```

## Architecture & Concepts
The Directionality class is an abstract contract within the world generation framework, specifically designed to control the orientation of generated structures, known as prefabs. It serves as a strategy pattern, encapsulating the logic for determining how a prefab should be rotated based on its surrounding environment.

This component acts as a decision-making engine. During world generation, when a system needs to place a prop or structure, it consults a Directionality implementation to calculate the appropriate rotation. For example, a concrete subclass could define logic to make a campfire prop always face away from a nearby cliff edge, or to align a bridge with the flow of a river.

A key architectural feature is the static factory method noDirectionality, which implements the Null Object Pattern. This provides a default, non-operative instance that returns no rotation and requires no scanning. This pattern is critical for simplifying the world generation code, allowing systems to treat all props uniformly without requiring constant null checks for props that have no specific orientation rules.

## Lifecycle & Ownership
- **Creation:** As an abstract class, Directionality is never instantiated directly. Concrete subclasses are instantiated by the world generator's configuration loader when parsing prop definitions. The special null-object instance is created on-demand via the static noDirectionality factory method.
- **Scope:** An instance of a Directionality subclass is tied to the lifecycle of the prop definition it belongs to. It is typically loaded once when the server initializes the world generator and persists for the entire server session. These objects are effectively stateless, long-lived configuration holders.
- **Destruction:** Instances are eligible for garbage collection when the world generator or its associated prop configurations are unloaded, such as during a server shutdown or a hot-reload of zone files.

## Internal State & Concurrency
- **State:** The abstract base class is stateless. Concrete implementations are expected to be immutable. Their behavior is defined by their class logic, not by mutable instance fields. They are designed to be pure functions that transform an input context into an output rotation.
- **Thread Safety:** This class and its implementations **must be thread-safe**. World generation is a massively parallel process, with multiple chunks being generated simultaneously by different worker threads. A single Directionality instance will be accessed concurrently by many threads to determine rotations for different locations. All implementations must be free of side effects and rely only on the provided Pattern.Context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRotationAt(Pattern.Context) | PrefabRotation | O(N) | Calculates the specific rotation for a prefab at a given world position. N is the number of blocks in the read range. Returns null if no specific rotation is determined. |
| getGeneralPattern() | Pattern | O(1) | Returns the abstract Pattern associated with this directionality rule. |
| getReadRangeWith(Scanner) | Vector3i | O(1) | Defines the 3D volume (bounding box) around a point that must be scanned to gather enough context to make a rotation decision. |
| getPossibleRotations() | List<PrefabRotation> | O(1) | Returns a complete list of all potential rotations that this rule could possibly produce. Used for validation and optimization. |
| noDirectionality() | static Directionality | O(1) | Factory method for the null-object implementation. |

## Integration Patterns

### Standard Usage
Directionality is not used directly by game logic but is a core component of the world generation pipeline. A prop placement service retrieves the appropriate Directionality rule from a prop's definition, uses it to determine rotation, and then applies that rotation during instantiation.

```java
// Pseudo-code for a prop placement system
PropDefinition prop = getPropForLocation(x, y, z);
Directionality directionality = prop.getDirectionality();

// The scanner provides the environmental data
Pattern.Context context = scanner.scan(x, y, z, directionality.getReadRangeWith(scanner));

// The core decision is made here
PrefabRotation rotation = directionality.getRotationAt(context);

// Place the prop with the calculated rotation
world.placePrefab(prop.getPrefab(), new Vector3i(x, y, z), rotation);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create subclasses that store state between calls to getRotationAt. Each calculation must be independent and deterministic based on the input context.
- **Expensive Computations:** Avoid performing computationally expensive operations, disk I/O, or network calls within getRotationAt. This method is on a hot path during world generation and must execute extremely quickly.
- **Ignoring Context:** Implementations should not return a random or hardcoded rotation without inspecting the provided Pattern.Context. The purpose of the system is to make environmentally-aware decisions.

## Data Pipeline
The Directionality class is a processing stage in the procedural generation of a single prop. It consumes local world data and produces a rotational output.

> Flow:
> Prop Placement Request -> Scanner reads local block data -> Pattern.Context created -> **Directionality.getRotationAt** -> PrefabRotation -> Prefab Instantiator -> Placed Prop in World State


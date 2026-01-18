---
description: Architectural reference for ConstantAssignments
---

# ConstantAssignments

**Package:** com.hypixel.hytale.builtin.hytalegenerator.propdistributions
**Type:** Transient

## Definition
```java
// Signature
public class ConstantAssignments extends Assignments {
```

## Architecture & Concepts
The ConstantAssignments class is a foundational implementation of the Assignments contract within the world generation's prop distribution system. It represents the simplest possible prop placement strategy: a deterministic, single-prop assignment.

Architecturally, it serves as a leaf node or a base case in the Strategy pattern employed by the prop generator. While other Assignments implementations might involve complex logic based on noise functions, biome data, or probabilistic tables, ConstantAssignments provides a fixed, non-variant outcome. It is used for regions, layers, or biomes where exactly one type of prop should be placed, without exception. Its primary purpose is to provide a low-overhead, predictable behavior for simple decoration or filling tasks.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, typically by higher-level configuration parsers or biome definition factories. It is not a managed service and should not be treated as one. A new instance is created for each rule that requires a constant prop assignment.
- **Scope:** The lifetime of this object is bound to the parent configuration that created it, such as a BiomeDefinition or a specific generation layer rule. It is short-lived relative to the application session.
- **Destruction:** The object is eligible for garbage collection as soon as its parent configuration is unloaded or falls out of scope. It manages no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Immutable. The state, consisting of the target Prop and its associated runtime cost, is provided at construction and is encapsulated in final fields. The object's behavior is guaranteed to be consistent throughout its lifetime.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature, an instance of ConstantAssignments can be safely shared and accessed by multiple world-generation worker threads concurrently without any need for external locking or synchronization. This is a critical feature for ensuring high performance in the parallelized world generator.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| propAt(position, id, distance) | Prop | O(1) | Returns the configured Prop instance. This method ignores all input parameters, providing a constant result. |
| getRuntime() | int | O(1) | Returns the pre-configured computational cost associated with this assignment. |
| getAllPossibleProps() | List<Prop> | O(1) | Returns an immutable list containing only the single Prop this object is configured with. |

## Integration Patterns

### Standard Usage
This class is intended to be used by a prop generation service or a biome processor. The processor holds a reference to an Assignments implementation and queries it to determine which prop to place at a given location.

```java
// A world generator obtains an Assignments object from a biome's configuration.
// In this case, the biome is configured to always place oak trees.
Assignments propRules = biome.getPropAssignments(); // Returns an instance of ConstantAssignments

// The generator queries for the prop at a specific world coordinate.
Prop propToPlace = propRules.propAt(new Vector3d(128, 64, 256), workerId, 0.0);

// The generator then proceeds to place the returned prop (the oak tree).
world.placeProp(propToPlace, new Vector3d(128, 64, 256));
```

### Anti-Patterns (Do NOT do this)
- **Misapplication for Variety:** Do not use ConstantAssignments in contexts where varied or probabilistic prop placement is required. Using this class will override any noise-based or rule-based variety, resulting in a monotonous distribution of a single prop.
- **State Modification:** Do not attempt to modify the internal Prop or runtime fields using reflection. The immutability of this class is a core design assumption for ensuring thread safety in the world generator.

## Data Pipeline
The data flow for this component is unidirectional and simple. It acts as a terminal provider in a request-response pattern initiated by the world generator.

> Flow:
> World Generator Request -> `propAt(position)` -> **ConstantAssignments** -> Returns configured Prop instance -> World Generator places Prop


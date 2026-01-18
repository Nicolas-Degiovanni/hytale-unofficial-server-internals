---
description: Architectural reference for PropField
---

# PropField

**Package:** com.hypixel.hytale.builtin.hytalegenerator
**Type:** Data Structure / Transient

## Definition
```java
// Signature
public class PropField {
```

## Architecture & Concepts
The PropField class is an immutable data structure that serves as a declarative instruction for the Hytale world generator. It encapsulates a complete "prop placement job" by combining spatial, content, and scheduling information into a single, portable object.

In the context of procedural generation, a "prop" refers to a decorative world object like a tree, rock, or bush. A PropField defines a region and the rules for populating it with such props. It does not perform the generation itself; rather, it is a configuration object consumed by a higher-level prop placement executor.

Its design combines three critical components:
1.  **PositionProvider:** Defines *where* props should be placed. This abstracts the spatial logic, allowing for various distribution strategies like random scattering in a volume, placement along a spline, or arrangement in a grid.
2.  **Assignments:** Defines *what* props to place. This object contains the rules for selecting specific prop assets, including their frequency, rotation, and scale variations.
3.  **runtime:** An integer identifier that dictates *when* this placement job should be executed relative to others. This allows the generator to orchestrate complex scenes by layering prop fields in a specific order, for example, placing large rocks before small pebbles.

By encapsulating these concerns, PropField acts as a fundamental building block for creating complex and varied environments.

### Lifecycle & Ownership
-   **Creation:** A PropField is instantiated directly via its public constructor, typically by a higher-level generator system (e.g., a BiomeGenerator or a StructureGenerator). These parent systems create PropField instances to define the decorative elements within their domain.
-   **Scope:** The object is transient and its lifetime is bound to a specific world generation task. It is created, passed to a prop placement service, consumed, and then becomes eligible for garbage collection. It does not persist beyond the generation of a given world chunk or region.
-   **Destruction:** Managed by the Java garbage collector. Once the world generation executor has finished processing the instruction, all references to the PropField are dropped.

## Internal State & Concurrency
-   **State:** **Immutable**. All fields are marked as final (implicitly, via constructor assignment with no setters) and are set only once during construction. The state of a PropField instance cannot be modified after it is created.

-   **Thread Safety:** **Inherently Thread-Safe**. Due to its immutability, a PropField instance can be safely shared and read by multiple worker threads simultaneously without locks or other synchronization primitives. This is a critical feature for enabling a highly parallelized world generation pipeline.

## API Surface
The public contract is minimal, consisting only of accessors for its constituent configuration objects.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPositionProvider() | PositionProvider | O(1) | Returns the provider responsible for determining prop locations. |
| getPropDistribution() | Assignments | O(1) | Returns the object defining which props to place and their distribution rules. |
| getRuntime() | int | O(1) | Returns the runtime phase identifier for scheduling the placement operation. |

## Integration Patterns

### Standard Usage
A PropField is intended to be created by a generator and submitted to a processing queue or executor. The executor then deconstructs the object to perform the actual work.

```java
// A generator constructs a PropField to define a forest area
PositionProvider forestArea = new BoxPositionProvider(...);
Assignments treeTypes = new WeightedAssignments("oak_tree", "birch_tree");
int generationPass = 10; // Run after terrain but before small foliage

PropField forestPropField = new PropField(generationPass, treeTypes, forestArea);

// The generator submits it to the world generation context
worldGenContext.getPropExecutor().submit(forestPropField);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Subclassing:** Do not extend PropField to introduce mutable state. The system relies on its immutability for thread safety and predictable generation. Modifying a PropField after its creation is a severe violation of its design contract.
-   **Misusing Runtime:** The runtime field is not a generic data field. It is specifically for ordering and phasing generation passes. Using it for other purposes, such as storing a prop count or a random seed, will lead to unpredictable and incorrect world generation.

## Data Pipeline
PropField acts as a structured data packet within the broader world generation pipeline. It represents the transition from abstract configuration to a concrete, executable task.

> Flow:
> Biome Configuration (JSON/HOCON) -> Deserializer -> **PropField Instance** -> Prop Placement Executor -> World Chunk Data


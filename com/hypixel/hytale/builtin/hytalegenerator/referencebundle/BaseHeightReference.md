---
description: Architectural reference for BaseHeightReference
---

# BaseHeightReference

**Package:** com.hypixel.hytale.builtin.hytalegenerator.referencebundle
**Type:** Transient

## Definition
```java
// Signature
public class BaseHeightReference extends Reference {
```

## Architecture & Concepts
The BaseHeightReference class is a foundational component within the procedural world generation framework. It functions as an immutable *value object* designed to encapsulate a terrain height generation algorithm.

Its primary architectural purpose is to decouple the world generator pipeline from the specific implementation of height calculation. By wrapping a standard BiDouble2DoubleFunction functional interface, it provides a strongly-typed, predictable contract for any system that needs to determine the base elevation of the terrain at a given 2D coordinate (x, z).

This class is not a service or a manager; it is a data carrier. It represents a piece of configuration—the "how" of height generation—that is passed into the various stages of the generator, such as biome placement, water level calculation, and structure spawning.

### Lifecycle & Ownership
- **Creation:** Instantiated directly by high-level configuration code, typically during the setup of a world generation profile or a specific biome definition. It is not managed by a dependency injection container or service locator.
- **Scope:** The lifetime of a BaseHeightReference instance is tied to a specific generation task. It is created, passed down through the generator call stack, and becomes eligible for garbage collection once the task is complete. It does not persist.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods associated with this class.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, the heightFunction, is a final field assigned only once during construction. The object's state cannot be modified after creation, making it inherently predictable.
- **Thread Safety:** **Thread-safe**. Due to its immutability, an instance of BaseHeightReference can be safely shared and accessed by multiple generator worker threads concurrently without any need for external synchronization.

**WARNING:** While the BaseHeightReference wrapper is thread-safe, the thread safety of the *contained function* is the responsibility of its implementer. Functions passed to the constructor should be pure and free of side effects to prevent concurrency bugs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHeightFunction() | BiDouble2DoubleFunction | O(1) | Retrieves the encapsulated height calculation function. This method is guaranteed to never return null. |

## Integration Patterns

### Standard Usage
The intended use is to define a height calculation algorithm as a lambda or method reference and encapsulate it within this class. This reference is then passed into a generator context or pipeline.

```java
// Define a simple flat plane height function using a lambda
BiDouble2DoubleFunction flatPlane = (x, z) -> 64.0;
BaseHeightReference heightRef = new BaseHeightReference(flatPlane);

// This reference is then consumed by a generator stage
terrainGenerator.setBaseHeight(heightRef);
terrainGenerator.processChunk(chunkCoords);
```

### Anti-Patterns (Do NOT do this)
- **Null Functions:** Passing a null function to the constructor violates the Nonnull contract and will lead to a NullPointerException, either immediately or during a later stage of world generation.
- **Stateful Functions:** Providing a stateful lambda or function object that modifies external state is a critical anti-pattern. This will introduce severe concurrency issues in a multi-threaded generator and result in non-deterministic, inconsistent world generation. Height functions must be pure.

## Data Pipeline
BaseHeightReference acts as a configuration input to the data pipeline, not as a processing stage itself. It provides the logic used by other stages.

> Flow:
> World Generation Profile -> **BaseHeightReference (Configuration)** -> Terrain Generation Stage -> Height Value Calculation -> Voxel Buffer Population


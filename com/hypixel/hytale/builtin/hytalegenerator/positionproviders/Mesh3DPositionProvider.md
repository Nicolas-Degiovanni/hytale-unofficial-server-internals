---
description: Architectural reference for Mesh3DPositionProvider
---

# Mesh3DPositionProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders
**Type:** Transient

## Definition
```java
// Signature
public class Mesh3DPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The **Mesh3DPositionProvider** is a specialized component within the procedural world generation framework. It functions as a high-level adapter, bridging the contract of the abstract **PositionProvider** with a concrete point generation strategy defined by a **PointProvider**.

Its primary architectural role is to translate a generic request for positions within a bounded volume (the **Context**) into a specific 3D point generation call. This implements the Strategy Pattern, allowing world generation logic to remain agnostic of the specific algorithm used to generate points (e.g., grid, random, Poisson disk). The **Mesh3DPositionProvider** itself does not contain any generation logic; it is a pure delegate.

This class is fundamental for populating 3D volumes with features, such as ore veins, underground structures, or particle effect locations, where the placement is determined by a configurable point cloud algorithm.

## Lifecycle & Ownership
-   **Creation:** **Mesh3DPositionProvider** instances are created during the setup phase of a world generation pass. They are typically instantiated by a factory or configuration loader that parses world generation rules and injects the appropriate **PointProvider** dependency.
-   **Scope:** The lifetime of an instance is ephemeral, scoped strictly to the execution of the specific generation task it was configured for. It does not persist between different generation passes or game sessions.
-   **Destruction:** The object is eligible for garbage collection as soon as the owning generation task completes and all references to it are released. There is no manual destruction or cleanup required.

## Internal State & Concurrency
-   **State:** The internal state consists of a single, final reference to a **PointProvider** instance, making the **Mesh3DPositionProvider** itself effectively immutable after construction. It does not cache results or maintain any mutable state across calls.
-   **Thread Safety:** This class is conditionally thread-safe. It introduces no synchronization mechanisms or mutable state of its own. Its safety in a multi-threaded world generation environment is entirely dependent on the thread safety of the **PointProvider** instance injected during construction.

    **WARNING:** Injecting a **PointProvider** that is not thread-safe can lead to severe data corruption, race conditions, or non-deterministic world generation when used in a parallelized generator.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Mesh3DPositionProvider(PointProvider) | constructor | O(1) | Constructs the provider, injecting the point generation strategy. |
| positionsIn(Context) | void | O(N) | Fulfills the **PositionProvider** contract by delegating to the injected **PointProvider**. Complexity is dependent on the injected strategy, where N is the number of points generated. |

## Integration Patterns

### Standard Usage
This component is not intended for direct use in game logic. It should be configured as part of a larger world generation definition and instantiated by the generator's bootstrap process.

```java
// Within a hypothetical world generator setup
// 1. Define the point generation strategy
PointProvider gridStrategy = new GridPointProvider(/* spacing */ 16);

// 2. Adapt the strategy using Mesh3DPositionProvider
PositionProvider positionSource = new Mesh3DPositionProvider(gridStrategy);

// 3. The generator uses the abstract provider to place features
worldGenerator.addFeature("hytale:gold_ore", positionSource);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Delegation:** Do not inject a **PointProvider** that relies on mutable state if the **Mesh3DPositionProvider** instance will be shared across different threads or generation chunks. This will break generation determinism.
-   **Incorrect Context:** While the class name implies 3D, the system relies on the injected **PointProvider**'s behavior. Using a **PointProvider** that only generates 2D points may lead to unexpected placement on a single plane.

## Data Pipeline
The **Mesh3DPositionProvider** acts as a simple pass-through component in the data pipeline, translating a high-level context into a direct call to a more primitive generator.

> Flow:
> World Generator Task -> `positionsIn(Context)` -> **Mesh3DPositionProvider** -> `pointGenerator.points3d()` -> `context.consumer.accept(position)`


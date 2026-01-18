---
description: Architectural reference for EnvironmentContainer
---

# EnvironmentContainer

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class EnvironmentContainer {
```

## Architecture & Concepts

The EnvironmentContainer is a fundamental component of the server-side world generation pipeline. It functions as a rule-based selection engine, responsible for determining which "environment"—conceptually a biome or a distinct geological region—should be generated at a specific world coordinate.

Architecturally, this class implements a **Chain of Responsibility** pattern. It holds an ordered list of EnvironmentContainerEntry objects, each representing a conditional rule for placing a specific set of environments. When queried for a coordinate, the container iterates through its list of entries. The first entry whose spatial condition is met is given control to select and return the final environment ID.

This design provides a powerful, data-driven way to define complex worlds. World generation behavior is not hardcoded but is instead defined in external configuration files that are deserialized into this container structure.

The core components of this system are:
*   **EnvironmentContainer:** The top-level object that orchestrates the selection process.
*   **EnvironmentContainerEntry:** A single rule containing a condition and a result mapping.
*   **ICoordinateCondition:** A predicate that returns true or false, determining if an entry's rule applies to a given (x, z) coordinate. This acts as the gatekeeper for the rule.
*   **IWeightedMap:** A data structure that maps noise values to environment IDs, allowing for varied environment placement even within a region governed by a single rule.
*   **NoiseProperty:** A noise function used to generate a value at the given coordinate, which is then fed into the IWeightedMap to perform the final selection.

A mandatory DefaultEnvironmentContainerEntry acts as a fallback, guaranteeing that an environment is always selected if no other conditional entry matches.

### Lifecycle & Ownership
-   **Creation:** EnvironmentContainer instances are not intended for manual instantiation during the game loop. They are created during the server's world initialization phase, typically by deserializing world generation profile files (e.g., JSON or HOCON) into this object graph.
-   **Scope:** The lifetime of an EnvironmentContainer is tied to the lifetime of the WorldGenerator that uses it. It persists as long as the world it defines is active on the server.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or the world is unloaded, and all references from the world generation system are released.

## Internal State & Concurrency
-   **State:** This class is effectively **immutable**. Its internal fields, defaultEntry and entries, are final and are assigned only once at construction. All constituent objects (like NoiseProperty and ICoordinateCondition) are also designed to be immutable. This makes the container a read-only data structure post-initialization.

-   **Thread Safety:** The EnvironmentContainer is **fully thread-safe**. Its immutability guarantees that multiple threads can call getEnvironmentAt concurrently without any risk of race conditions or data corruption. This is a critical design feature that enables the engine to parallelize chunk generation across multiple worker threads, significantly improving world generation performance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEnvironmentAt(seed, x, z) | int | O(N) | Determines the environment ID for a given world coordinate. Iterates through N entries. Throws no exceptions. |

## Integration Patterns

### Standard Usage
The EnvironmentContainer is a core dependency for any system that performs biome or environment placement. It is retrieved from the world's configuration and used to query for environment IDs on a per-column basis.

```java
// Within a WorldGenerator or similar system
// container is typically a member field initialized from world config

public Biome getBiomeForColumn(int x, int z) {
    int worldSeed = this.world.getSeed();
    int environmentId = this.environmentContainer.getEnvironmentAt(worldSeed, x, z);
    return BiomeRegistry.getById(environmentId);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new EnvironmentContainer()` in gameplay or procedural logic. These objects represent static world definitions and must be loaded from configuration. Manually creating them can lead to worlds that are inconsistent with their defined profiles.
-   **Stateful Conditions:** Avoid creating and using a custom ICoordinateCondition implementation that relies on mutable external state. Doing so would break the thread-safety guarantee of the container and lead to severe, difficult-to-debug concurrency issues during parallel chunk generation.

## Data Pipeline
The EnvironmentContainer sits at the heart of the environment selection stage within the larger world generation process. Its data flows from static configuration files through to the final block placement logic.

> Flow:
> World Profile (JSON) -> Deserializer -> **EnvironmentContainer** instance -> WorldGenerator.getEnvironmentAt(seed, x, z) -> Environment ID -> Biome Placement System -> Final Chunk Data

---
# EnvironmentContainer.EnvironmentContainerEntry

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Transient Data Structure

## Definition
```java
// Signature
public static class EnvironmentContainerEntry {
```

## Architecture & Concepts
EnvironmentContainerEntry is the atomic unit of logic within the EnvironmentContainer. It represents a single, self-contained rule for environment placement. Each entry binds a spatial condition to a noise-driven selection mechanism.

The core responsibility of an entry is twofold:
1.  **Applicability Check:** Through its ICoordinateCondition, it first determines if it is even relevant for a given (x, z) coordinate. This allows for rules that only apply to specific continents, temperature zones, or other large-scale procedural features.
2.  **Environment Selection:** If the condition is met, it uses its NoiseProperty to sample a value at the coordinate. This value is then passed to its IWeightedMap, which translates the noise value into a concrete environment ID. This two-step process allows for both broad, conditional placement (the condition) and fine-grained, textured variation within that placement (the noise and map).

### Lifecycle & Ownership
-   **Creation:** Instantiated alongside its parent EnvironmentContainer during deserialization of world configuration files.
-   **Scope:** Its lifetime is identical to and strictly managed by its parent EnvironmentContainer.
-   **Destruction:** Garbage collected when its parent container is collected.

## Internal State & Concurrency
-   **State:** This class is **immutable**. All its fields are final and assigned at construction.
-   **Thread Safety:** Inherently **thread-safe** due to its immutable design. The shouldGenerate and getEnvironmentAt methods are pure functions with no side effects, making them safe for concurrent execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldGenerate(seed, x, z) | boolean | O(C) | Evaluates the internal ICoordinateCondition. Complexity C depends on the condition's implementation. |
| getEnvironmentAt(seed, x, z) | int | O(N) | Samples the NoiseProperty and uses the IWeightedMap to select an environment ID. Complexity N depends on the map's implementation. |

## Integration Patterns

### Standard Usage
This class is not used directly. It is an internal component of the EnvironmentContainer. Developers interact with it by defining its properties in world generation configuration files, not by calling its methods from code.

### Anti-Patterns (Do NOT do this)
-   **Method Invocation:** Do not call shouldGenerate or getEnvironmentAt directly. Always go through the parent EnvironmentContainer, which correctly manages the chain of responsibility and the default fallback case. Bypassing the container can lead to missing environments or incorrect rule prioritization.


---
description: Architectural reference for CompositeInt2Flags
---

# CompositeInt2Flags

**Package:** com.hypixel.hytale.server.worldgen.util.condition.flag
**Type:** Transient

## Definition
```java
// Signature
public class CompositeInt2Flags implements Int2FlagsCondition {
```

## Architecture & Concepts
The CompositeInt2Flags class is a specialized rule engine within the server-side world generation framework. Its primary function is to translate a single integer input, such as terrain height or a noise value, into a final integer bitmask representing a set of flags.

Architecturally, it implements a **Chain of Responsibility** pattern. An instance of this class holds an ordered sequence of *FlagCondition* rules. When the *eval* method is invoked, it processes this chain sequentially, allowing each rule to conditionally modify a running integer value. This design enables the declarative construction of complex, layered logic for assigning properties to world features.

For example, a world generator can define a set of rules:
1.  If input (height) is above 128, apply the MOUNTAIN flag.
2.  If input (height) is above 128 AND another condition (temperature) is low, also apply the SNOWY flag.

This component is critical for decoupling world generation algorithms from the specific rules that define biome characteristics, material placement, or other flag-based metadata.

## Lifecycle & Ownership
-   **Creation:** Instances are typically created during the server's bootstrap phase by configuration loaders that parse world generation definitions from asset files (e.g., JSON). They are not intended for dynamic creation during the game loop.
-   **Scope:** The object is a stateless value object. Its lifetime is tied to the world generation configuration that defines it. It generally persists for the entire server session as part of a larger, immutable ruleset.
-   **Destruction:** Managed by the Java garbage collector. When the server shuts down or a world is unloaded, the configuration objects holding references to CompositeInt2Flags instances are released, making them eligible for collection. No explicit destruction logic is required.

## Internal State & Concurrency
-   **State:** The object's state, consisting of the *defaultResult* and the array of *FlagCondition* objects, is **effectively immutable**. The fields are final and are assigned only once at construction. The *eval* method is a pure function with no side effects on the object's state.
-   **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable nature, a single instance can be safely shared and accessed by multiple world generation threads simultaneously without locks or synchronization. This is a crucial feature for enabling parallelized chunk generation.

## API Surface
The public contract is minimal, exposing only the evaluation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int input) | int | O(N) | Evaluates the input against the internal chain of N FlagCondition rules. Returns the final integer bitmask. |

## Integration Patterns

### Standard Usage
This class should be treated as a pre-configured component. A higher-level system, such as a BiomeProvider, would hold an instance and invoke it to determine flags for a given world coordinate or data point.

```java
// Assume 'heightCondition' is a pre-loaded CompositeInt2Flags instance
// representing rules based on terrain height.
int terrainHeight = getTerrainHeightAt(x, z);

// Evaluate the height to get a bitmask of flags (e.g., IS_MOUNTAIN, IS_WATER)
int terrainFlags = heightCondition.eval(terrainHeight);

// Use the flags for subsequent generation steps
if ((terrainFlags & IS_MOUNTAIN) != 0) {
    placeMountainFoliage();
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid constructing this class manually in game logic. Instances should be defined as static data in configuration files and loaded at startup. This separates world generation rules from engine code.
-   **Stateful Conditions:** The underlying *IIntCondition* implementations passed to *FlagCondition* must also be thread-safe and preferably immutable. A stateful condition would violate the thread-safety guarantees of the entire composite structure.
-   **Excessively Long Chains:** While the system is efficient, rule chains with thousands of conditions can create performance hotspots in tight world generation loops. For highly complex logic, consider alternative data structures like lookup tables or pre-baked data.

## Data Pipeline
The data flow is a linear pipeline that transforms a single integer into a final bitmask.

> Flow:
> World Gen Input (e.g., Noise Value) -> **CompositeInt2Flags.eval()** -> [FlagCondition 1 -> FlagCondition 2 -> ... -> FlagCondition N] -> Final Integer Flag Set


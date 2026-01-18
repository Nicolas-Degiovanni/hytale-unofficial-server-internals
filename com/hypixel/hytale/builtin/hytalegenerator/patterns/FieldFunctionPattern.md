---
description: Architectural reference for FieldFunctionPattern
---

# FieldFunctionPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class FieldFunctionPattern extends Pattern {
```

## Architecture & Concepts
The FieldFunctionPattern is a fundamental component within the procedural world generation framework. It acts as a bridge between continuous mathematical data, represented by a Density function, and discrete placement decisions within the world. Its primary purpose is to evaluate a scalar field (like noise, elevation, or temperature) at a specific world coordinate and determine if the resulting value falls within one or more predefined numerical ranges.

This mechanism allows for the creation of complex, organic, and non-grid-aligned structures. For example, a FieldFunctionPattern can be configured to "match" all locations where a 3D Perlin noise function returns a value between 0.5 and 0.6. This effectively carves out cavern systems, defines the boundaries of an ore vein, or determines the shoreline of a lake.

It implements the Pattern contract, signaling its role as a predicate in a larger generation system. Multiple patterns are often combined to define the complex logic for biomes, structures, and geological features.

### Lifecycle & Ownership
- **Creation:** Instantiated programmatically by a higher-level generator or biome definition during its configuration phase. It is constructed with a mandatory Density object that serves as its input function.
- **Scope:** The lifetime of a FieldFunctionPattern instance is tied to its parent configuration. It is not a global or session-scoped object. It persists as long as the generator that uses it is active, typically for the duration of a specific world generation task.
- **Destruction:** The object is eligible for garbage collection once the world generation process that created it completes and all references to it are released.

## Internal State & Concurrency
- **State:** **Mutable**. The internal state is defined by its Density field and its internal list of Delimiter ranges. The public API includes the method addDelimiter, which directly mutates this internal list. Therefore, an instance of FieldFunctionPattern is considered fully configured only after all necessary delimiters have been added.

- **Thread Safety:** **Conditionally Safe**. The class is **not thread-safe for mutation**. Calling addDelimiter from one thread while another thread calls matches will result in undefined behavior, likely a ConcurrentModificationException.

    **WARNING:** All configuration via addDelimiter **must** be completed before the pattern is used by the multithreaded world generation engine.

    Once configuration is complete, the matches method is read-only and is safe to be called concurrently by multiple worker threads, as it does not modify any internal state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Pattern.Context context) | boolean | O(D + N) | The core evaluation method. Calculates density via the internal Density function and checks it against N delimiters. Complexity D is determined by the Density function. |
| readSpace() | SpaceSize | O(1) | Returns the spatial volume this pattern needs to read to perform its evaluation. |
| addDelimiter(double min, double max) | void | O(1) | **Mutates state.** Adds a new valid numerical range. This method is not thread-safe. |

## Integration Patterns

### Standard Usage
The FieldFunctionPattern is designed to be instantiated, configured with one or more ranges, and then passed to a generator which will invoke the matches method across many different world coordinates.

```java
// 1. Obtain a density function (e.g., 3D noise)
Density noiseField = NoiseFunctions.getPerlin3D();

// 2. Create the pattern with the density function
FieldFunctionPattern cavePattern = new FieldFunctionPattern(noiseField);

// 3. Configure the pattern by adding valid ranges (delimiters)
// This will match any point where the noise value is between 0.4 and 0.6
cavePattern.addDelimiter(0.4, 0.6);

// 4. Pass the fully configured pattern to a generator system
// The generator will now use cavePattern.matches(context) to decide where to place air blocks.
worldGenerator.addFeature(Blocks.AIR, cavePattern);
```

### Anti-Patterns (Do NOT do this)
- **Post-Initialization Mutation:** Do not retain a reference to a FieldFunctionPattern and call addDelimiter on it after it has been submitted to the world generator. This will cause severe race conditions when worker threads begin calling the matches method. All configuration must be completed upfront.

- **Empty Delimiter List:** Instantiating a FieldFunctionPattern and failing to add any delimiters is a common logical error. In this state, the matches method will **always** return false, as there are no valid ranges to check against.

## Data Pipeline
The flow of data through this component is linear and transforms a spatial coordinate into a boolean decision.

> Flow:
> Pattern.Context (World Coordinates) -> Density.process() -> `double` (Scalar Value) -> **FieldFunctionPattern.matches()** -> `boolean` (Match/No-Match) -> World Generator Logic


---
description: Architectural reference for DensityDelimitedEnvironmentProvider
---

# DensityDelimitedEnvironmentProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.environmentproviders
**Type:** Stateful Component

## Definition
```java
// Signature
public class DensityDelimitedEnvironmentProvider extends EnvironmentProvider {
```

## Architecture & Concepts
The DensityDelimitedEnvironmentProvider is a fundamental routing component within the procedural world generation system. It acts as a high-level conditional selector, determining which specific environment should be generated at a given world coordinate.

Its core responsibility is to translate a continuous floating-point *density value* (often derived from a noise function like Perlin or Simplex noise) into a discrete environmental outcome. It achieves this by evaluating the density value against a pre-configured list of ranges, called Delimiters. Each Delimiter maps a specific numerical range to another, more specialized, EnvironmentProvider.

In essence, this class implements a Strategy pattern. It does not generate environmental data itself but instead delegates the final responsibility to a subordinate EnvironmentProvider based on the input from a Density function. This pattern allows for the powerful layering of generation logic, where broad, low-frequency density functions can define large-scale features like biomes, and this class acts as the bridge to select the appropriate detailed generator for that biome.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor, typically during the parsing and loading of world generation configuration files (e.g., a biome definition JSON). It is not managed by a dependency injection container or a central service registry.
-   **Scope:** The object's lifetime is bound to the specific world generator configuration that created it. It persists in memory as part of a larger, composite generator object graph.
-   **Destruction:** Marked for garbage collection when the parent world generator configuration is unloaded or replaced.

## Internal State & Concurrency
-   **State:** The internal state consists of a list of DelimiterDouble objects and a reference to a Density object. This state is provided at construction and is treated as **immutable** for the lifetime of the object. The constructor performs a shallow copy of the provided delimiters into its internal list.

-   **Thread Safety:** This class is **conditionally thread-safe**. The `getValue` method is re-entrant and does not modify any internal state. Concurrency safety is therefore dependent on the thread safety of the injected `Density` and nested `EnvironmentProvider` dependencies. In standard Hytale world generation, these dependencies are designed to be thread-safe to support parallel chunk generation, making this class safe for concurrent use in that context. No internal locking mechanisms are used.

## API Surface
The public contract is minimal, focused on construction and the primary evaluation method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DensityDelimitedEnvironmentProvider(delimiters, density) | constructor | O(N) | Constructs the provider. Filters the incoming list to exclude delimiters with invalid ranges (min >= max). |
| getValue(context) | int | O(N) | Evaluates the density at the given context and returns the value from the first matching delimited provider. Returns 0 if no range matches. |

*N = number of delimiters*

## Integration Patterns

### Standard Usage
This class is intended to be composed as part of a larger generator definition. The user configures a set of rules (delimiters) that map density outputs to specific environment generators.

```java
// 1. Define the underlying providers for different outcomes
EnvironmentProvider oceanProvider = new StaticEnvironmentProvider(OCEAN_ID);
EnvironmentProvider plainsProvider = new StaticEnvironmentProvider(PLAINS_ID);
EnvironmentProvider mountainProvider = new StaticEnvironmentProvider(MOUNTAIN_ID);

// 2. Define the ranges and associate them with providers
List<DelimiterDouble<EnvironmentProvider>> delimiters = List.of(
    new DelimiterDouble<>(new RangeDouble(-1.0, 0.2), oceanProvider),
    new DelimiterDouble<>(new RangeDouble(0.2, 0.6), plainsProvider),
    new DelimiterDouble<>(new RangeDouble(0.6, 1.0), mountainProvider)
);

// 3. Define the density function (e.g., a noise function)
Density elevationDensity = new PerlinNoiseDensity(...);

// 4. Construct the router
EnvironmentProvider biomeSelector = new DensityDelimitedEnvironmentProvider(delimiters, elevationDensity);

// 5. Use it during world generation
int biomeId = biomeSelector.getValue(new EnvironmentProvider.Context(x, y, z));
```

### Anti-Patterns (Do NOT do this)
-   **Overlapping Ranges:** The system evaluates delimiters in list order and returns on the first match. Defining overlapping ranges (e.g., [0.0, 0.5] and [0.4, 0.8]) will cause the second rule to be partially or completely shadowed, leading to non-deterministic or incorrect generation. The class does not validate against this.
-   **Gaps in Ranges:** Failing to define delimiters for all possible outputs of the Density function can create gaps. Any density value falling into such a gap will cause `getValue` to return the default of 0, which may be an unintended outcome (e.g., rendering as "air" or a default stone block).
-   **Empty Delimiter List:** Constructing this class with an empty list of delimiters will cause `getValue` to *always* return 0.

## Data Pipeline
The flow of data through this component is linear and synchronous. It acts as a transformation and routing step within a larger generation sequence.

> Flow:
> EnvironmentProvider.Context -> **DensityDelimitedEnvironmentProvider** (gets Density) -> Density Function -> double (density value) -> **DensityDelimitedEnvironmentProvider** (finds matching Delimiter) -> Nested EnvironmentProvider -> int (environment ID)


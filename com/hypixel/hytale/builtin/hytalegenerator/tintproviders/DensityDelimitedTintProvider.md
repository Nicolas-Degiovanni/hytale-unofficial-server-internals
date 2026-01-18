---
description: Architectural reference for DensityDelimitedTintProvider
---

# DensityDelimitedTintProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.tintproviders
**Type:** Transient

## Definition
```java
// Signature
public class DensityDelimitedTintProvider extends TintProvider {
```

## Architecture & Concepts

The DensityDelimitedTintProvider is a fundamental component within the procedural world generation engine, responsible for selecting a color tint based on environmental data. It functions as a conditional router or a data-driven selector, implementing a form of the **Strategy Pattern**.

Its core purpose is to evaluate a continuous value, known as a *density* (e.g., temperature, humidity, or elevation derived from a noise function), and map it to a specific, discrete TintProvider. This is achieved by defining a series of non-overlapping numerical ranges, each associated with a child TintProvider.

This architecture decouples the source of the environmental data from the logic that determines the final color. It enables world designers to create complex, layered biome transitions by defining rules such as: "If the temperature density is between 0.0 and 0.3, use the TundraTintProvider; if it is between 0.3 and 0.7, use the ForestTintProvider."

## Lifecycle & Ownership

-   **Creation:** Instances are not intended for manual creation in game logic code. They are instantiated by the world generator's configuration loader when parsing world generation asset files (e.g., JSON definitions). The list of delimiters and the associated Density function are injected during construction.
-   **Scope:** The object's lifetime is bound to the loaded world generation configuration. It is an effectively immutable, read-only data object that persists as long as the generator definition is active.
-   **Destruction:** The object is eligible for garbage collection when the world generator configuration is unloaded, for instance, when a server shuts down or a different world is loaded.

## Internal State & Concurrency

-   **State:** The internal state, consisting of the list of DelimiterDouble objects and the Density function, is **effectively immutable**. This state is established exclusively within the constructor and is not modified during the object's lifetime. The class is therefore stateless with respect to its processing operations.
-   **Thread Safety:** This class is **thread-safe** and designed for high-concurrency environments. Its read-only internal state allows the getValue method to be safely invoked by multiple world generation worker threads simultaneously without locks or synchronization. This guarantee assumes that the injected Density and child TintProvider instances are also thread-safe, which is a core contract of the world generation framework.

## API Surface

The public contract is minimal, exposing only the core processing method inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue(Context context) | TintProvider.Result | O(N) | Calculates a density value from the provided context and linearly scans N delimiters to find a match. If a range contains the value, it delegates to the associated provider. Returns WITHOUT_VALUE if no range matches. |

## Integration Patterns

### Standard Usage

This class is not used directly by gameplay programmers. It is a configuration primitive used by world designers within data files. The world generation engine invokes it as part of the block and biome tinting pipeline.

```java
// This class is configured via data files, not instantiated directly in code.
// The following is a conceptual representation of its usage by the world generator.

// 1. Configuration (Loaded from a world generator definition file)
List<DelimiterDouble<TintProvider>> biomeRules = List.of(
    new DelimiterDouble<>(new RangeDouble(0.0, 0.5), coldRegionTintProvider),
    new DelimiterDouble<>(new RangeDouble(0.5, 1.0), warmRegionTintProvider)
);
Density temperatureDensityFunction = new PerlinNoiseDensity(...);

// The engine constructs the provider from this data
TintProvider selector = new DensityDelimitedTintProvider(
    biomeRules,
    temperatureDensityFunction
);

// 2. Usage by the World Generation Engine during chunk creation
TintProvider.Context generationContext = new TintProvider.Context(world, position);
TintProvider.Result finalTint = selector.getValue(generationContext);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new DensityDelimitedTintProvider()` in application logic. This component's behavior should be defined entirely within world generation data assets to allow for hot-reloading and designer iteration.
-   **Overlapping Ranges:** Defining delimiters with overlapping ranges is a critical design flaw. The implementation will return the **first** match it finds. This makes the order of delimiters significant and can produce non-obvious or incorrect behavior that is difficult to debug. Ranges should be contiguous and non-overlapping.
-   **Stateful Delegates:** Do not inject Density or TintProvider implementations that are not thread-safe. This will violate the concurrency guarantees of the system and lead to data corruption or race conditions during parallel chunk generation.

## Data Pipeline

The component acts as a selection node in a larger data processing graph. It transforms a general context into a specific request for a downstream provider.

> Flow:
> WorldGen Context -> Density Function -> **DensityDelimitedTintProvider** -> Selected Child TintProvider -> Final Tint Result


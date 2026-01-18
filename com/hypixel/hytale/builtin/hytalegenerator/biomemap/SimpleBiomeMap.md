---
description: Architectural reference for SimpleBiomeMap
---

# SimpleBiomeMap

**Package:** com.hypixel.hytale.builtin.hytalegenerator.biomemap
**Type:** Transient

## Definition
```java
// Signature
public class SimpleBiomeMap<V> extends BiomeMap<V> {
```

## Architecture & Concepts
The SimpleBiomeMap is a concrete implementation of the abstract BiomeMap. It serves as a configurable data provider within the world generation pipeline, specifically for biome placement and transition information.

Its primary architectural role is to decouple the raw biome generation logic from the rules that govern how those biomes blend together. It achieves this by wrapping a BiCarta instance, which acts as the true source of biome data for any given coordinate. SimpleBiomeMap then decorates this raw data with metadata, such as transition radii, which are consumed by later stages of the world generator (e.g., terrain smoothing and decoration placement).

This class does not generate biomes itself; it is a stateful container and delegate for a more complex underlying system represented by the BiCarta interface. The generic type parameter V is inherited from the base BiomeMap and is not utilized by the methods within this specific implementation, but is maintained for compatibility with the broader framework.

## Lifecycle & Ownership
- **Creation:** An instance of SimpleBiomeMap is created by a higher-level world generation controller or stage manager. It is constructed with a mandatory BiCarta dependency, which must be fully initialized before the SimpleBiomeMap is created.
- **Scope:** The lifetime of a SimpleBiomeMap instance is typically bound to a single, complete world generation task. It is not a global singleton and should not be persisted across sessions.
- **Destruction:** The object is eligible for garbage collection as soon as the world generation process that owns it completes and releases its reference. There are no native resources requiring explicit cleanup.

## Internal State & Concurrency
- **State:** SimpleBiomeMap is a **mutable** object. It maintains internal state for biome transition rules, including a default transition radius and a map of radii for specific biome pairs. This state can be modified after instantiation.

- **Thread Safety:** This class is **not thread-safe** for mutation. The internal state, particularly the pairHashToRadius map, is not protected by synchronization. Concurrent calls to configuration methods like setDefaultRadius from multiple threads will lead to undefined behavior.

  However, the primary data access method, apply, is conditionally thread-safe. It delegates its call to the underlying BiCarta. If the provided BiCarta instance is thread-safe, then the apply method can be safely called from multiple worker threads simultaneously, as is intended by the WorkerIndexer.Id parameter.

  **WARNING:** All configuration of a SimpleBiomeMap instance must be completed in a single-threaded context before it is passed to parallel worker threads for biome sampling.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setDefaultRadius(int defaultRadius) | void | O(1) | Sets the default transition size between any two biomes. Throws IllegalArgumentException for non-positive values. |
| apply(int x, int z, WorkerIndexer.Id id) | BiomeType | Delegated | Retrieves the definitive BiomeType for a given world coordinate. Complexity is determined by the injected BiCarta. |
| allPossibleValues() | List<BiomeType> | Delegated | Returns a list of all biomes that the underlying BiCarta can possibly generate. |

## Integration Patterns

### Standard Usage
The intended use involves instantiating the class with a pre-configured BiCarta, setting any custom transition rules, and then passing it to world generation workers for read-only biome lookups.

```java
// 1. Obtain a fully configured BiCarta from a generator stage
BiCarta<BiomeType> biomeSource = generatorContext.getBiomeCarta();

// 2. Create the map, injecting the source
SimpleBiomeMap biomeMap = new SimpleBiomeMap(biomeSource);

// 3. Configure transition properties in a single-threaded context
biomeMap.setDefaultRadius(8);
// biomeMap.setRadiusForPair(BiomeA, BiomeB, 12); // Example of other potential configs

// 4. Pass the configured map to worker threads for read-only access
worldGenExecutor.submit(() -> {
    BiomeType currentBiome = biomeMap.apply(x, z, workerId);
    // ... process biome
});
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never modify the map's radii from one thread while other threads might be calling the apply method. This is a severe race condition and will lead to unpredictable generation artifacts.
- **Misunderstanding Delegation:** Do not attempt to modify the SimpleBiomeMap to change the fundamental biome layout. This class only stores metadata *about* the layout. The biome at (x, z) is determined solely by the injected BiCarta.

## Data Pipeline
SimpleBiomeMap acts as both a proxy for biome data and a source of configuration for subsequent pipeline stages.

> **Data Lookup Flow:**
> World Coordinate (x, z) -> **SimpleBiomeMap.apply()** -> BiCarta.apply() -> Final BiomeType

> **Configuration Consumption Flow:**
> **SimpleBiomeMap** (holds transition radii) -> Terrain Smoothing Stage -> Reads radii to calculate blending weights -> Smoothed Terrain Output


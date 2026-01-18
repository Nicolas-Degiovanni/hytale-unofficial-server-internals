---
description: Architectural reference for the MaterialSource interface, a core contract in the world generation system.
---

# MaterialSource

**Package:** com.hypixel.hytale.builtin.hytalegenerator.biome
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface MaterialSource {
```

## Architecture & Concepts
The MaterialSource interface is a fundamental contract within the Hytale world generation framework. It establishes a standardized abstraction for any object that can supply materials for terrain generation, most commonly a Biome.

By defining a single method, getMaterialProvider, this interface decouples the high-level world generator from the specific material selection logic of a given biome or terrain feature. This employs the **Strategy Pattern**, allowing different biomes to implement varied and complex rules for material placement without altering the core generation orchestrator.

An object implementing MaterialSource signals its role as a foundational component in the terrain composition pipeline. The generator queries this interface to retrieve the specific MaterialProvider strategy, which is then executed to determine the final Material (e.g., stone, dirt, grass) for a given block coordinate.

### Lifecycle & Ownership
As an interface, MaterialSource itself has no lifecycle. The lifecycle described here pertains to its concrete implementations.

-   **Creation:** Implementations are typically instantiated by the world generation service during the loading and configuration of biome definitions. These definitions are often deserialized from asset files.
-   **Scope:** An instance of a MaterialSource implementation persists for the lifetime of the world generator's configuration. It is effectively a stateless configuration object.
-   **Destruction:** The object is eligible for garbage collection when the world generator is shut down or reconfigured, and all references to the biome definitions are released.

## Internal State & Concurrency
-   **State:** The interface is stateless. Implementations are expected to be immutable or effectively immutable after initialization. Their primary role is to hold a reference to a configured MaterialProvider instance.
-   **Thread Safety:** This contract provides no inherent thread safety guarantees; safety is the responsibility of the implementing class. As world generation is a heavily multi-threaded process, all implementations of MaterialSource **must be thread-safe**. The returned MaterialProvider must also be safe for concurrent access by multiple chunk generation worker threads.

## API Surface
The public contract consists of a single method designed for high-frequency access during terrain composition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMaterialProvider() | MaterialProvider<Material> | O(1) | Retrieves the material selection strategy for this source. This method is expected to be a non-blocking, fast field accessor. |

## Integration Patterns

### Standard Usage
The interface is not used directly but is implemented by configuration objects like a Biome. The world generator retrieves the provider from the biome corresponding to a specific world coordinate.

```java
// Assume 'currentBiome' is the biome at a given coordinate and implements MaterialSource
MaterialSource materialSource = (MaterialSource) currentBiome;

// Retrieve the strategy for material selection
MaterialProvider<Material> provider = materialSource.getMaterialProvider();

// Use the provider to get the final material for a block
Material finalMaterial = provider.get(world, x, y, z);
```

### Anti-Patterns (Do NOT do this)
-   **Null Return:** Implementations of getMaterialProvider must never return null. A null return will cause a fatal NullPointerException deep within the world generation pipeline, leading to chunk generation failures. Always provide a default or fallback MaterialProvider.
-   **Expensive Computation:** Do not perform heavy calculations, disk I/O, or any blocking operations within the getMaterialProvider method. It is called frequently and must return immediately. All complex logic should be deferred to the returned MaterialProvider instance.

## Data Pipeline
MaterialSource acts as a factory or gateway in the data flow of material selection during world generation. It does not process data itself but rather provides the next component in the chain.

> Flow:
> World Generator -> Biome (as MaterialSource) -> **getMaterialProvider()** -> MaterialProvider Strategy -> Material Selection Logic -> Final Material


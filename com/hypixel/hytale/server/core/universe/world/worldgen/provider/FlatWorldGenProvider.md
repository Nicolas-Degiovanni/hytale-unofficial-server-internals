---
description: Architectural reference for FlatWorldGenProvider
---

# FlatWorldGenProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen.provider
**Type:** Transient Factory

## Definition
```java
// Signature
public class FlatWorldGenProvider implements IWorldGenProvider {
```

## Architecture & Concepts
The FlatWorldGenProvider is a concrete implementation of the **IWorldGenProvider** strategy pattern. Its primary role is to serve as a deserializable configuration object and factory for a simple, flat-world generator. It is not the world generator itself, but rather the blueprint used to construct one.

This class embodies a critical design principle in Hytale's world generation system: the separation of configuration from execution.

1.  **Configuration Phase:** A FlatWorldGenProvider instance is created by the server's Codec system, typically by parsing a world configuration file. At this stage, it holds human-readable data, such as block names ("Soil_Grass") and environment names.

2.  **Baking Phase:** The `getGenerator` method is invoked. This method acts as a "baking" or "compilation" step. It validates the configuration and, most importantly, resolves the string-based asset names into their optimized integer ID representations by querying the global BlockType and Environment asset maps.

3.  **Execution Phase:** The result of `getGenerator` is an instance of the private inner class **FlatWorldGen**, which implements the high-performance IWorldGen interface. This lean, optimized object contains only the integer IDs and logic necessary for generation and is used repeatedly by the server to generate chunks on demand.

This two-stage process ensures that expensive and fallible operations like string lookups and validation happen only once when the world is loaded, not during the performance-critical task of chunk generation.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's world loading system during server startup. The static **CODEC** field is used to deserialize a configuration block (e.g., from a JSON file) into a FlatWorldGenProvider object. It is never intended to be instantiated manually via its constructor in game logic.
-   **Scope:** Its lifetime is exceptionally short. It exists only during the world initialization sequence.
-   **Destruction:** After the `getGenerator` method is called and the resulting IWorldGen instance is passed to the world, the FlatWorldGenProvider object is no longer referenced and becomes eligible for garbage collection. It does not persist for the life of the server.

## Internal State & Concurrency
-   **State:** The object is highly mutable during its creation via the Codec system. Its fields are populated directly during deserialization. The `getGenerator` method further mutates the internal state of its Layer objects by populating the `blockId` and `environmentId` fields. After this method is called, the object's state should be considered final.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and used in a single-threaded context during the server's initialization phase. Concurrent access, especially during the `getGenerator` call, will lead to race conditions and unpredictable behavior. The object it *produces*, FlatWorldGen, is thread-safe for generation as its state is read-only.

## API Surface
The public API is defined by the IWorldGenProvider interface. The primary entry point is `getGenerator`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGenerator() | IWorldGen | O(L) | Validates the layer configuration and resolves all asset names to integer IDs. Creates and returns a new, optimized FlatWorldGen instance. Throws WorldGenLoadException if configuration is invalid or assets are not found. L is the number of layers. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game-logic developers. It is driven by server configuration files. The following example shows how a world configuration file would specify this provider.

```yaml
# Example world.json configuration snippet
# The server's world loader parses this and uses the CODEC
# to create a FlatWorldGenProvider instance.

worldgen: {
  provider: "Flat",
  config: {
    Tint: { r: 91, g: 158, b: 40 },
    Layers: [
      { From: 0, To: 1, BlockType: "Bedrock" },
      { From: 1, To: 60, BlockType: "Stone", Environment: "Cave" },
      { From: 60, To: 64, BlockType: "Soil_Dirt" },
      { From: 64, To: 65, BlockType: "Soil_Grass", Environment: "Plains" }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new FlatWorldGenProvider()`. The world generation system is data-driven. To create a flat world, modify the world's configuration files.
-   **State Mutation After Use:** Do not modify the provider's fields after calling `getGenerator`. The returned IWorldGen instance is a separate object with a snapshot of the configuration at the time of its creation; it will not reflect subsequent changes to the provider.
-   **Reusing Providers:** Do not call `getGenerator` multiple times on a single instance. This is inefficient as it will re-run validation and asset resolution. The provider's lifecycle is create, use once, and discard.

## Data Pipeline
The FlatWorldGenProvider is a key component in the configuration-to-execution data pipeline for world generation.

> Flow:
> World Configuration File (JSON) -> Server Codec System -> **FlatWorldGenProvider Instance** -> `getGenerator()` Invocation -> **FlatWorldGen Instance** -> World Chunk Request -> GeneratedChunk


---
description: Architectural reference for IWorldGenProvider
---

# IWorldGenProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen.provider
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IWorldGenProvider {
```

## Architecture & Concepts
The IWorldGenProvider interface is a foundational contract within the server's world generation pipeline. It embodies the **Factory** and **Strategy** design patterns to decouple the world management system from the concrete implementations of world generators.

Its primary role is to provide an abstraction layer for retrieving an IWorldGen instance. This allows the server to support multiple, distinct world generation algorithms and to select the appropriate one dynamically based on configuration.

The static CODEC field is a critical component, indicating that the system is data-driven. The engine uses this `BuilderCodecMapCodec` to deserialize world configuration files (e.g., JSON). When a world is loaded, its configuration specifies a generator "Type", which the CODEC uses to look up and instantiate the corresponding IWorldGenProvider implementation. This makes the world generation system highly extensible, as new generator types can be registered with the codec without changing the core world loading logic.

In essence, this interface is the entry point for configuring *how* a world is generated, acting as a factory for the object that performs the actual generation.

### Lifecycle & Ownership
As an interface, IWorldGenProvider itself has no lifecycle. The following applies to its concrete implementations.

- **Creation:** Implementations are instantiated by the serialization system via the static CODEC field during the loading of a world's configuration data. The engine does not create these objects directly; their creation is a result of deserializing a data file.
- **Scope:** An instance of an IWorldGenProvider implementation is scoped to the world it is configured for. It persists as long as the parent world object is held in memory by the server's Universe.
- **Destruction:** The object is eligible for garbage collection when the world it belongs to is unloaded and all references to it are released. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The interface contract is stateless. However, concrete implementations may be stateful. They might cache the created IWorldGen instance or hold configuration parameters deserialized from the world data.
- **Thread Safety:** The interface itself provides no thread safety guarantees. Implementations are not guaranteed to be thread-safe. It is expected that a single provider instance is used within the context of a single world's generation pipeline. Accessing a provider from multiple threads without external synchronization is a severe anti-pattern and will lead to undefined behavior.

**WARNING:** World generation is a highly parallelized process. Do not assume a provider instance or the generator it returns can be safely shared across different chunk generation threads unless the specific implementation explicitly guarantees it.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGenerator() | IWorldGen | Variable | Retrieves the configured world generator. May create a new instance or return a cached one. Throws WorldGenLoadException on critical failure. |

## Integration Patterns

### Standard Usage
The provider is not intended for direct use by most game logic. It is an internal component of the world loading system. A higher-level manager retrieves the provider during world initialization and uses it to obtain the generator.

```java
// Conceptual example of engine-level usage
// WorldConfig is deserialized, which implicitly creates the provider
IWorldGenProvider provider = worldConfig.getGenProvider();

try {
    IWorldGen generator = provider.getGenerator();
    // ... use generator to create chunks
} catch (WorldGenLoadException e) {
    // Handle catastrophic world load failure
    log.error("Failed to get world generator for world: " + world.getName(), e);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never manually instantiate a concrete implementation (e.g., `new OverworldGenProvider()`). The entire system is designed to be driven by the CODEC and world configuration files. Direct instantiation bypasses configuration loading and will result in an improperly initialized generator.
- **Ignoring Exceptions:** The `getGenerator` method can throw a WorldGenLoadException. This is a fatal error indicating that the world cannot be generated as configured. This exception must be caught and handled, typically by preventing the world from loading.
- **State Assumption:** Do not assume `getGenerator` returns a new instance every time. Implementations may cache and return the same instance. Do not modify the state of a returned IWorldGen object unless you are certain of the provider's caching strategy.

## Data Pipeline
The IWorldGenProvider is a key step in the transformation of static configuration data into a live, procedural world generator.

> Flow:
> World Configuration File (JSON) -> Server Codec Engine -> **IWorldGenProvider (Instantiation)** -> WorldManager calls getGenerator() -> IWorldGen (Instance) -> Chunk Generation System


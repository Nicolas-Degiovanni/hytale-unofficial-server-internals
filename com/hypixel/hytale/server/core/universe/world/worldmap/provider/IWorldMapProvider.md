---
description: Architectural reference for IWorldMapProvider
---

# IWorldMapProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.provider
**Type:** Provider Interface

## Definition
```java
// Signature
public interface IWorldMapProvider {
```

## Architecture & Concepts
The IWorldMapProvider interface defines an abstract factory contract for creating or retrieving an IWorldMap instance. Its primary role is to decouple the World entity from the concrete mechanisms of world map generation and loading. This allows for multiple strategies—such as procedural generation, loading from disk, or network streaming—to be implemented and selected at runtime without altering the core World logic.

This interface is a key component of the server's polymorphic world data system. The static CODEC field, a BuilderCodecMapCodec, is a critical architectural element. It facilitates the serialization and deserialization of world configuration files. The system uses a string identifier, "Type", to look up and instantiate the correct concrete IWorldMapProvider implementation, enabling flexible and data-driven world setup.

## Lifecycle & Ownership
As an interface, IWorldMapProvider itself does not have a lifecycle. The lifecycle pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated by the server's configuration loader during world initialization. The static CODEC is used to deserialize world settings and create the appropriate provider instance based on the configuration data.
- **Scope:** An instance of an IWorldMapProvider implementation is scoped to a single World. It generally persists for the entire lifetime of that World object.
- **Destruction:** The provider instance is eligible for garbage collection when the associated World is unloaded and all references to it are released.

## Internal State & Concurrency
- **State:** The interface contract is stateless. However, concrete implementations may be stateful. For example, a provider for a pre-generated world might cache the IWorldMap instance after the first load to improve performance on subsequent calls. A procedural provider might be stateless, creating a new generator each time.
- **Thread Safety:** This interface makes no guarantees about thread safety. Implementations are responsible for ensuring their own thread safety if they are to be accessed from multiple threads, such as during asynchronous chunk generation.

**WARNING:** Implementations that cache data or manage shared resources must implement appropriate synchronization mechanisms. The caller should not assume the provider is thread-safe.

## API Surface
The public contract consists of a single method for generation and a static field for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGenerator(World world) | IWorldMap | Implementation-dependent | Retrieves or constructs the IWorldMap for the given World. Throws WorldMapLoadException on failure. |
| CODEC | BuilderCodecMapCodec | O(1) | Static codec used by the engine to serialize and deserialize provider configurations. |

## Integration Patterns

### Standard Usage
The provider is typically defined in a world configuration file and deserialized by the server engine. The engine then uses the provider to supply the World with its map during the initialization phase.

```java
// Engine-level code during world loading
// Provider is obtained via deserialization, not direct instantiation
IWorldMapProvider provider = worldConfiguration.getMapProvider();

try {
    // The World object receives its map generator from the provider
    IWorldMap mapGenerator = provider.getGenerator(world);
    world.setWorldMap(mapGenerator);
} catch (WorldMapLoadException e) {
    // Handle critical world load failure
    log.error("Failed to initialize world map for: " + world.getName(), e);
}
```

### Anti-Patterns (Do NOT do this)
- **Implementation Casting:** Do not cast an IWorldMapProvider to a concrete type. This violates the abstraction and couples game logic to a specific map generation strategy, making the system brittle.
- **Ignoring Exceptions:** Never swallow a WorldMapLoadException. A failure to provide a world map is a critical error that leaves the World in an invalid state and must be handled properly.
- **State Assumption:** Do not assume `getGenerator` will return the same instance every time. The contract does not guarantee this; behavior is implementation-specific.

## Data Pipeline
The IWorldMapProvider acts as a factory in the data and control flow of world creation. It translates configuration data into a functional game component.

> Flow:
> World Configuration File -> Server Deserializer (using CODEC) -> **IWorldMapProvider Instance** -> World Initialization Service -> `getGenerator()` call -> Live IWorldMap Object


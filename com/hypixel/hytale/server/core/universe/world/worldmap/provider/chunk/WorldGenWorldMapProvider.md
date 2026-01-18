---
description: Architectural reference for WorldGenWorldMapProvider
---

# WorldGenWorldMapProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.provider.chunk
**Type:** Transient

## Definition
```java
// Signature
public class WorldGenWorldMapProvider implements IWorldMapProvider {
```

## Architecture & Concepts

The WorldGenWorldMapProvider is a specialized implementation of the IWorldMapProvider interface that acts as a **delegating proxy**, not a direct generator. Its primary role is to bridge the world's core terrain generator (IWorldGen) with the world map system. It does not contain any map generation logic itself.

This provider implements a form of the **Chain of Responsibility** pattern. When asked for a map generator, it first queries the World's configured IWorldGen instance. If that IWorldGen instance also implements IWorldMapProvider, this class delegates the request to it. This allows custom world generators to supply their own specialized world map implementations.

If the configured IWorldGen does not provide its own map, this class acts as a fallback, returning a default, generic ChunkWorldMap instance. This ensures that the system always receives a valid IWorldMap, preventing failures from misconfigured or simple world generators.

**WARNING:** The class name can be misinterpreted. It does not *generate* a map; it *retrieves* the map provider *from* the primary world generator. The `toString` method, which returns "DisabledWorldMapProvider", more accurately reflects its nature as a non-operational proxy.

## Lifecycle & Ownership

-   **Creation:** Instantiated by the server's configuration system during world initialization. The static `CODEC` field is used by a higher-level service to construct an instance from world configuration data, typically a JSON or similar file. It is not intended for manual instantiation.
-   **Scope:** Short-lived and transient. An instance is typically created to fulfill a single request to resolve the correct IWorldMap for a given World. It holds no state and can be garbage collected immediately after use.
-   **Destruction:** Handled by standard Java garbage collection. No explicit cleanup methods are required or provided.

## Internal State & Concurrency

-   **State:** **Stateless and Immutable.** This class contains no instance fields and its behavior is entirely determined by the arguments passed to its methods.
-   **Thread Safety:** **Fully Thread-Safe.** As a stateless object, a single instance can be safely shared and invoked from multiple threads without synchronization. Thread safety of the overall operation depends on the atomicity of the `world.getChunkStore().getGenerator()` call and the thread safety of the provider it may delegate to.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGenerator(World world) | IWorldMap | O(1) | Resolves and returns the appropriate IWorldMap for the given World. Delegates to the world's IWorldGen if possible, otherwise returns a default. |

## Integration Patterns

### Standard Usage

This provider is not used directly by feature developers. It is specified within a world's configuration and invoked by the core world-loading services to acquire a map generator.

```java
// Conceptual example of how the server might use this provider
// A developer would NOT write this code.

// 1. Provider is loaded from world config via its CODEC
IWorldMapProvider provider = worldConfig.getMapProvider(); // Returns an instance of WorldGenWorldMapProvider

// 2. The World uses the provider to get the actual map generator
IWorldMap mapGenerator = provider.getGenerator(thisWorld);

// 3. The map generator is then used by other systems
mapSystem.initialize(mapGenerator);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new WorldGenWorldMapProvider()`. The server relies on the `CODEC` mechanism to manage provider lifecycles and configuration. Direct instantiation bypasses this system and will likely fail in a production environment.
-   **Ignoring the Fallback:** Relying on this provider to always return a custom map from your IWorldGen can mask configuration errors. If your IWorldGen fails to implement IWorldMapProvider correctly, the system will silently fall back to the basic ChunkWorldMap, which may cause unexpected behavior in systems that require a more advanced map.

## Data Pipeline

This component acts as a factory in a control flow rather than a step in a data processing pipeline. Its function is to resolve a dependency for the world system.

> Flow:
> World Configuration Loaded -> **WorldGenWorldMapProvider** (Instantiated via CODEC) -> `getGenerator()` called by World -> Delegation to `IWorldGen` -> Returns `IWorldMap` instance


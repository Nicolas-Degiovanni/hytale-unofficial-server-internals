---
description: Architectural reference for DisabledWorldMapProvider
---

# DisabledWorldMapProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.provider
**Type:** Stateless Provider (Null Object Pattern)

## Definition
```java
// Signature
public class DisabledWorldMapProvider implements IWorldMapProvider {
```

## Architecture & Concepts
The DisabledWorldMapProvider is a specific implementation of the IWorldMapProvider strategy interface. Its primary architectural role is to serve as a **Null Object**. This pattern provides a non-functional, safe-to-use object that conforms to the required interface, thereby eliminating the need for explicit null checks in the calling code.

When a server administrator disables the world map feature in the world configuration, the engine's configuration loader selects and instantiates this provider. Any system component that subsequently requests a world map generator will receive a valid IWorldMap object that performs no operations and immediately returns empty results. This design greatly simplifies the map generation logic, as it can operate on any IWorldMap instance without special-casing for a disabled state.

The presence of a static CODEC field indicates that this provider is part of a larger, data-driven system where provider implementations are chosen and deserialized based on a configuration key, in this case, the static ID "Disabled".

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's world configuration system during the world loading phase. The selection is driven by configuration data that is deserialized using the static CODEC. It is not intended for manual instantiation.
- **Scope:** The DisabledWorldMapProvider instance itself is typically transient, existing only long enough for the world loader to call getGenerator. The IWorldMap instance it returns is a static singleton (DisabledWorldMap.INSTANCE) that persists for the lifetime of the Java Virtual Machine.
- **Destruction:** The provider object is eligible for garbage collection once the world has retrieved the singleton IWorldMap generator from it. The returned singleton is never destroyed.

## Internal State & Concurrency
- **State:** This class is **completely stateless and immutable**. It contains no instance fields and its behavior is constant. The inner DisabledWorldMap class is also stateless.
- **Thread Safety:** This provider is inherently **thread-safe**. Due to its stateless nature, it can be safely accessed by any number of threads without requiring synchronization. All methods that return a CompletableFuture provide an instance that is already completed, ensuring non-blocking and predictable behavior.

## API Surface
The public contract is minimal, exposing only the factory method required by the IWorldMapProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGenerator(World world) | IWorldMap | O(1) | Returns a singleton, no-op IWorldMap implementation. This call is guaranteed to be non-blocking and will never throw a WorldMapLoadException. |

## Integration Patterns

### Standard Usage
A developer or system component should never interact with this class directly. It is intended to be managed exclusively by the world configuration and loading systems. The correct pattern is to retrieve the configured provider and treat it polymorphically.

```java
// Correct interaction is via the interface, driven by configuration.
// The system, not the developer, chooses DisabledWorldMapProvider.

IWorldMapProvider configuredProvider = world.getConfig().getWorldMapProvider();
IWorldMap mapGenerator = configuredProvider.getGenerator(world);

// This call is safe and will complete instantly with an empty map
// if the configuredProvider is a DisabledWorldMapProvider.
CompletableFuture<WorldMap> futureMap = mapGenerator.generate(...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new DisabledWorldMapProvider()`. The provider system relies on the static CODEC for instantiation from configuration. Manual creation bypasses the intended design.
- **Type Checking:** Avoid checking if a provider is an instance of DisabledWorldMapProvider. The purpose of the Null Object pattern is to allow all IWorldMapProvider implementations to be treated identically. Code that checks for this specific type is brittle and violates the pattern.
- **Expecting Real Data:** Do not assume the IWorldMap returned by this provider will ever produce map tiles or points of interest. Systems using the generator must be robust enough to handle an empty, zero-dimension WorldMap.

## Data Pipeline
This provider acts as a terminal node in the data pipeline, effectively short-circuiting the map generation process and ensuring a predictable, empty output.

> Flow:
> World Configuration File -> Server World Loader -> **DisabledWorldMapProvider** -> getGenerator() -> DisabledWorldMap Singleton -> Map Generation System -> Instantly returns `CompletableFuture<Empty WorldMap>` -> **Pipeline Terminates**


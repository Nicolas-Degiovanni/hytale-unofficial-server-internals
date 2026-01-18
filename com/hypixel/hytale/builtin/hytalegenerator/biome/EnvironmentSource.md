---
description: Architectural reference for EnvironmentSource
---

# EnvironmentSource

**Package:** com.hypixel.hytale.builtin.hytalegenerator.biome
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface EnvironmentSource {
```

## Architecture & Concepts
The EnvironmentSource interface is a fundamental contract within the world generation system. It establishes a standardized mechanism for any world-generation component, most commonly a Biome, to expose its associated environmental data.

This interface acts as an abstraction layer, decoupling systems that need to query environmental parameters (e.g., biome placement logic, foliage decorators) from the concrete implementations that define those parameters. By conforming to this contract, an object signals its ability to provide an EnvironmentProvider, which contains the detailed logic for calculating values like temperature, humidity, and weirdness at any given point in the world.

Its primary role is to ensure that the complex, multi-threaded world generator can reason about the environmental characteristics of different regions in a uniform and predictable way.

### Lifecycle & Ownership
As an interface, EnvironmentSource does not have its own lifecycle. The lifecycle described here pertains to the objects that *implement* this interface.

-   **Creation:** Implementing objects, such as Biome definitions, are typically instantiated once during the world generator's bootstrap phase. This is often driven by loading configuration files or data assets.
-   **Scope:** An instance of an EnvironmentSource implementation is expected to be a long-lived, stateless object that persists for the entire duration of a world generation session.
-   **Destruction:** Instances are garbage collected when the world generation context is destroyed, for example, when a server shuts down or a new world is created with a different ruleset.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. Implementations are **strongly expected** to be immutable. The EnvironmentProvider returned by the contract method should be configured at creation time and never change. Modifying the environmental properties of a source at runtime would lead to non-deterministic and inconsistent world generation.
-   **Thread Safety:** **CRITICAL:** All implementations of EnvironmentSource **must be unconditionally thread-safe**. The world generator executes biome placement and feature decoration across multiple worker threads simultaneously. The getEnvironmentProvider method will be called concurrently from these threads and must not rely on or modify shared mutable state. It should be a simple, non-blocking field accessor.

## API Surface
The public contract is minimal, focused exclusively on retrieving the associated provider.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEnvironmentProvider() | EnvironmentProvider | O(1) | Retrieves the provider that defines environmental parameters for this source. This method must be non-blocking and thread-safe. |

## Integration Patterns

### Standard Usage
The most common pattern is for a system service, like a BiomeSelector, to query a potential biome for its environmental data to determine if it is a suitable candidate for a specific location.

```java
// A world-gen system receives a potential biome for placement.
// The 'biome' object is assumed to implement EnvironmentSource.
public boolean canPlaceBiome(EnvironmentSource biome, WorldCoordinate coord) {
    EnvironmentProvider provider = biome.getEnvironmentProvider();

    // Use the provider to make decisions
    float temperature = provider.getTemperatureAt(coord.x, coord.y, coord.z);
    if (temperature < COLD_THRESHOLD) {
        return false; // Biome is not suitable for this cold location
    }

    return true;
}
```

### Anti-Patterns (Do NOT do this)
-   **Lazy Initialization:** Do not perform lazy initialization or heavy computation within the getEnvironmentProvider method. This can introduce unpredictable performance stalls and potential race conditions in the highly parallelized generator. The provider should be created and assigned during the object's construction.
-   **Returning Null:** The getEnvironmentProvider method must never return null. A valid, non-null EnvironmentProvider is a core guarantee of this contract. Returning null will result in a NullPointerException downstream and crash the world generation worker thread.
-   **Mutable Implementations:** An object that implements EnvironmentSource but allows its underlying EnvironmentProvider to be changed after construction is a severe violation of design principles. This will break the assumption of deterministic world generation.

## Data Pipeline
EnvironmentSource is not part of a data processing pipeline but rather a query point for static configuration data.

> Flow:
> World Generation Service -> Queries Biome Object (as **EnvironmentSource**) -> Method getEnvironmentProvider() -> Returns EnvironmentProvider -> Service uses provider for calculations


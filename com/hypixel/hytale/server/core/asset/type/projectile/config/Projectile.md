---
description: Architectural reference for Projectile
---

# Projectile

**Package:** com.hypixel.hytale.server.core.asset.type.projectile.config
**Type:** Asset Data Object

## Definition

```java
// Signature
@Deprecated
public class Projectile implements JsonAssetWithMap<String, DefaultAssetMap<String, Projectile>>, BallisticData {
```

**Warning:** This class is marked as Deprecated and may be removed or replaced in future versions. Its usage should be limited to maintaining legacy systems.

## Architecture & Concepts

The Projectile class is a data-driven configuration object that serves as a blueprint for all projectile types within the game engine. It is not an active, in-world entity but rather an immutable template that defines the physical and behavioral properties of a projectile, such as an arrow or a fireball.

Its primary architectural role is within the Hytale Asset System. Instances of this class are deserialized from JSON definition files at server startup. The static `CODEC` field, an instance of AssetBuilderCodec, is the central mechanism for this process. This codec declaratively defines the mapping from JSON keys to the Java fields of the Projectile class.

Key architectural features include:

*   **Declarative Deserialization:** The extensive fluent builder chain on the static `CODEC` field defines how to parse, validate, and construct a Projectile object. This pattern decouples the data definition from the game logic.
*   **Asset Inheritance:** The codec is configured with `appendInherited`, allowing projectile definitions in JSON to form a hierarchy. A specific projectile (e.g., "hytale:fire_arrow") can inherit base properties from a parent (e.g., "hytale:base_arrow") and override only what is necessary.
*   **Cross-Asset Integration:** The codec includes validators that reference other asset systems. For example, it validates that the `appearance` field corresponds to a valid ModelAsset and that sound event IDs like `hitSoundEventId` point to existing SoundEvent assets. This ensures data integrity across the entire asset ecosystem.
*   **Physics Contract:** By implementing the BallisticData interface, this class provides a stable contract for physics systems, such as the SimplePhysicsProvider, to retrieve all necessary parameters for simulating projectile motion.

## Lifecycle & Ownership

The lifecycle of a Projectile object is strictly controlled by the server's asset management framework.

*   **Creation:** Instances are instantiated exclusively by the AssetStore during the server's initial asset loading phase. The static `CODEC` reads a JSON file, uses the private constructor to create a new Projectile object, and populates its fields. Direct instantiation by developers is prohibited.
*   **Scope:** Once loaded, a Projectile instance is stored in a static, global AssetMap and persists for the entire duration of the server session. It is treated as a globally accessible, read-only configuration constant.
*   **Destruction:** All loaded Projectile instances are de-referenced and eligible for garbage collection only when the AssetRegistry is shut down or reloaded, typically when the server stops.

## Internal State & Concurrency

*   **State:** The state of a Projectile object is **effectively immutable** after it is fully loaded and processed. While its fields are not declared as final, they are set only once during the deserialization process. After loading, the `processConfig` method is invoked to perform a final post-processing step, converting string-based asset IDs (e.g., `hitSoundEventId`) into more performant integer indices (e.g., `hitSoundEventIndex`). This derived data is then cached within the object.

*   **Thread Safety:** The object is **thread-safe for read operations**. Because its state is fixed after initialization, multiple game systems (e.g., physics, combat, audio) can safely access its properties from different threads without requiring locks or other synchronization primitives.

    **Warning:** The static `getAssetStore` method contains a lazy initialization block that is not inherently thread-safe. Accessing this method from multiple threads before the AssetRegistry is fully initialized can lead to race conditions. The engine mitigates this by ensuring all asset loading occurs within a single-threaded bootstrap phase before any multi-threaded game systems are started.

## API Surface

The public API is designed for retrieving projectile configurations and their properties. It is not intended for modification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, singleton AssetStore for all Projectile assets. |
| getAssetMap() | static DefaultAssetMap | O(1) | Returns a map of all loaded Projectile configurations, keyed by their string ID. |
| getId() | String | O(1) | Returns the unique asset identifier for this projectile configuration. |
| getMuzzleVelocity() | double | O(1) | Retrieves the initial speed of the projectile. |
| getGravity() | double | O(1) | Retrieves the gravitational acceleration applied to the projectile. |
| getDamage() | int | O(1) | Retrieves the base damage dealt upon impact. |
| getHitParticles() | WorldParticle | O(1) | Returns the particle effect configuration to spawn on successful hit. |
| getExplosionConfig() | ExplosionConfig | O(1) | Returns the explosion configuration, if any, triggered on impact or death. |

## Integration Patterns

### Standard Usage

The standard pattern is to retrieve a pre-loaded Projectile configuration from the global AssetMap using its known ID. This configuration object is then passed to systems responsible for spawning and managing entities.

```java
// How a developer should normally use this
DefaultAssetMap<String, Projectile> projectileMap = Projectile.getAssetMap();
Projectile arrowConfig = projectileMap.get("hytale:standard_arrow");

if (arrowConfig != null) {
    // Pass the configuration to the entity spawning system
    // or the physics engine to simulate its trajectory.
    double velocity = arrowConfig.getMuzzleVelocity();
    int damage = arrowConfig.getDamage();
    // ...
}
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Never use `new Projectile()`. The object is not designed for manual construction and will be incomplete. Always retrieve instances from the `Projectile.getAssetMap()`.
*   **State Modification:** Do not attempt to modify the fields of a retrieved Projectile instance via reflection or other means. These objects are shared globally, and modifying one will cause unpredictable behavior across the entire server.
*   **Premature Access:** Do not call `getAssetStore()` or `getAssetMap()` before the server's asset loading phase is complete. This will result in null pointers or incompletely populated asset maps.

## Data Pipeline

The Projectile class is the final product of a data transformation pipeline that begins with raw asset files on disk.

> Flow:
> JSON file (`assets/hytale/projectiles/arrow.json`) -> AssetStore Loader -> **Projectile.CODEC** (Deserialization, Inheritance, Validation) -> **Projectile Instance** -> `processConfig()` (Post-processing) -> Stored in global `AssetMap` -> Read by Game Systems (Physics, Combat)


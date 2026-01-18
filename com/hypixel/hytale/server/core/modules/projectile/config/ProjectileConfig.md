---
description: Architectural reference for ProjectileConfig
---

# ProjectileConfig

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Configuration Asset

## Definition
```java
// Signature
public class ProjectileConfig
   implements JsonAssetWithMap<String, DefaultAssetMap<String, ProjectileConfig>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ProjectileConfig>,
   BallisticData {
```

## Architecture & Concepts

The ProjectileConfig class is a server-side data container that defines the complete set of properties for a type of projectile. It serves as an immutable blueprint, deserialized from JSON asset files at server startup. This class is a cornerstone of Hytale's data-driven design, allowing developers and content creators to define new projectiles without recompiling the engine.

Architecturally, it sits at the intersection of several core systems:

*   **Asset System:** It is fundamentally an asset, managed by the global AssetRegistry. Its static CODEC field provides a declarative and robust mechanism for parsing JSON, applying validation rules, and handling inheritance between different projectile definitions.
*   **Physics Engine:** By implementing the BallisticData interface, it provides a standardized contract for the physics simulation to calculate trajectories based on properties like gravity and muzzle velocity.
*   **Networking:** Its implementation of NetworkSerializable allows it to be converted into a compact, client-safe packet format. This is critical for synchronizing projectile visuals and basic properties to players without exposing server-only configuration details.
*   **Game Logic:** Systems responsible for spawning projectiles (e.g., weapon handlers) query the AssetStore to retrieve a ProjectileConfig instance, which they then use to initialize a new Projectile entity in the world.

The static CODEC is the most complex and important feature. It uses a builder pattern to define how each JSON key maps to a Java field, what validation to perform (e.g., ensuring a referenced Model asset exists), and what post-processing logic to run.

### Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the AssetStore during the server's initial asset loading phase. The static CODEC field dictates the entire instantiation and population process from a corresponding JSON file. Direct instantiation is an anti-pattern and will result in a non-functional object.
-   **Scope:** An instance of ProjectileConfig persists for the entire server session. Once loaded, all configurations are cached in a static AssetStore and are treated as immutable.
-   **Destruction:** All instances are dereferenced and eligible for garbage collection when the server shuts down and the AssetRegistry is cleared.

## Internal State & Concurrency

-   **State:** The object's state is considered immutable after the post-deserialization hook, processConfig, is executed. This method pre-computes and caches integer indices for sound events, an optimization to avoid repeated string lookups at runtime. The getModel method employs lazy initialization, caching the generated Model object on its first invocation.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be loaded and processed by a single thread during server startup and subsequently read by the main game loop thread.
    -   **Warning:** The lazy initialization within the getModel method presents a race condition if accessed concurrently before the model is cached.
    -   **Warning:** The static getAssetStore method contains a check-then-act race condition on the ASSET_STORE field. Under the engine's single-threaded asset loading model, this is safe. However, any attempt to parallelize asset loading would require synchronization on this method.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | **Static.** Retrieves the central repository for all ProjectileConfig assets. |
| getAssetMap() | DefaultAssetMap | O(1) | **Static.** A convenience method to access the underlying map of all loaded projectile configs. |
| getModel() | Model | O(N) first call, O(1) after | Lazily generates and caches a unit-scale Model from the configured model asset. Not thread-safe. |
| getCalculatedOffset(pitch, yaw) | Vector3d | O(1) | Calculates the world-space spawn offset based on the shooter's orientation. |
| toPacket() | ProjectileConfig | O(N) | Serializes the configuration into a network packet for client synchronization. N is the number of interactions. |
| getPhysicsConfig() | PhysicsConfig | O(1) | Returns the detailed physics properties for this projectile. |
| getLaunchForce() | double | O(1) | Returns the initial velocity applied to the projectile upon launch. |

## Integration Patterns

### Standard Usage

A game system, such as a weapon handler, retrieves a pre-loaded configuration from the asset map and uses it to configure a new projectile entity.

```java
// Retrieve the config for a specific arrow type
ProjectileConfig arrowConfig = ProjectileConfig.getAssetMap().getAsset("hytale:arrow_standard");

if (arrowConfig != null) {
    // The ProjectileFactory would use this config to spawn and initialize a new entity
    world.getProjectileFactory().create(player, arrowConfig);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ProjectileConfig()`. The object will be uninitialized and lack critical data. Always retrieve instances from the static `getAssetStore()` or `getAssetMap()` methods.
-   **Runtime Modification:** Do not modify the state of a ProjectileConfig object after it has been loaded. The system treats these objects as immutable, and runtime changes will lead to unpredictable and inconsistent behavior across the server.
-   **Off-Thread Access:** Avoid accessing these objects from worker threads, especially methods with lazy initialization like `getModel()`, without external synchronization. They are intended for use within the main server game loop.

## Data Pipeline

The ProjectileConfig is a critical link in the chain that transforms a static data file into a dynamic in-game object.

> Flow:
> `hytale:arrow.json` -> AssetStore Deserializer (using `ProjectileConfig.CODEC`) -> **ProjectileConfig Instance** (in memory) -> Game Logic (e.g., Bow fires) -> Projectile Entity (spawned in world) -> `toPacket()` -> Network Packet -> Client Rendering
---


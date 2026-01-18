---
description: Architectural reference for PlayerConfig
---

# PlayerConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Asset / Data Transfer Object

## Definition
```java
// Signature
public class PlayerConfig {
```

## Architecture & Concepts
The PlayerConfig class is a data-driven asset that defines the physical and gameplay properties of a player entity. It is not an active component with logic, but rather a passive data structure that acts as a blueprint. Its primary architectural role is to bridge human-readable configuration files (e.g., JSON) with the server's high-performance runtime systems.

The core of this class is the static **CODEC** field, a `BuilderCodec`. This codec declaratively defines the entire serialization, validation, and post-processing pipeline for player configuration data.

Key architectural patterns include:

*   **Declarative Deserialization:** The `CODEC` specifies how to map keys from a data source to fields in a PlayerConfig instance. This decouples the data format from the class structure.
*   **Identifier to Index Resolution:** The system uses string-based identifiers (e.g., "BuiltinDefault") in configuration files for readability. During the `afterDecode` lifecycle hook, these identifiers are resolved into integer indices. This is a critical performance optimization, allowing runtime systems like the movement and collision engines to perform fast array lookups instead of expensive string comparisons or map lookups.
*   **Inheritance and Validation:** The use of `appendInherited` and `addValidator` indicates a sophisticated, hierarchical asset system. Configs can inherit and override values from parents, and the system enforces data integrity by validating that referenced configurations (like HitboxCollisionConfig) actually exist.

This class is a terminal node in the asset loading pipeline, providing finalized, engine-ready data to gameplay systems.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale asset loading system via the defined `CODEC`. The `PlayerConfig::new` constructor reference is invoked internally by the codec during deserialization of a corresponding asset file. **WARNING:** Manual instantiation will result in a non-functional object.
-   **Scope:** An instance of PlayerConfig is loaded for each unique player configuration file defined in the game assets. These instances are cached and managed by a central asset registry and persist for the entire lifetime of the server.
-   **Destruction:** Instances are garbage collected when the server shuts down and the asset registry is cleared.

## Internal State & Concurrency
-   **State:** The object's state is highly mutable during the deserialization and `afterDecode` phase. After being loaded and cached by the asset system, it must be treated as **effectively immutable**. The internal state contains both the original string identifiers from the asset file and the resolved integer indices used by the engine at runtime.
-   **Thread Safety:** This class is **not thread-safe for mutation**. It contains no internal locking mechanisms. It is, however, safe for concurrent reads *after* it has been fully constructed, post-processed, and published by the asset management system. Any external modification to a cached PlayerConfig instance after loading will lead to catastrophic and difficult-to-diagnose race conditions.

## API Surface
The public API is designed for read-only access to the resolved, engine-optimized configuration values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHitboxCollisionConfigIndex() | int | O(1) | Returns the resolved integer index for the player's hitbox configuration. |
| getRepulsionConfigIndex() | int | O(1) | Returns the resolved integer index for the player's entity repulsion configuration. |
| getMovementConfigIndex() | int | O(1) | Returns the resolved integer index for the player's movement physics configuration. |
| getMovementConfigId() | String | O(1) | Returns the original string identifier for the movement configuration. |
| getMaxDeployableEntities() | int | O(1) | Returns the maximum number of deployable entities a player can own simultaneously. |

## Integration Patterns

### Standard Usage
PlayerConfig is not typically queried directly. Instead, a player entity will hold a reference to its configured instance, and game systems will query that entity.

```java
// A game system (e.g., MovementSystem) processing a player entity
void applyPlayerMovement(PlayerEntity player) {
    // Retrieve the config from the player, not a global registry
    PlayerConfig config = player.getConfig();

    // Use the pre-resolved index for fast lookups
    int movementIndex = config.getMovementConfigIndex();
    MovementProfile profile = MovementProfileRegistry.getProfile(movementIndex);

    // Apply physics based on the loaded profile
    player.getPhysicsState().applyProfile(profile);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new PlayerConfig()`. This bypasses the entire `CODEC` pipeline, including default value assignment, validation, and the critical `afterDecode` logic. The resulting object will have uninitialized index fields (`-1` or `0`), leading to `ArrayIndexOutOfBoundsException` or incorrect default behavior in downstream systems.
-   **Using String IDs at Runtime:** Avoid using `getMovementConfigId()` in performance-critical game loops. The purpose of the `afterDecode` hook is to perform the expensive string-to-integer lookup once at load time. Always prefer the `get...Index()` methods in systems that run every tick.

## Data Pipeline
The PlayerConfig object is the result of a multi-stage data transformation pipeline that begins with a raw asset file on disk.

> Flow:
> Asset File (e.g., player.json) -> AssetLoader Service -> **PlayerConfig CODEC** (Deserialization & Validation) -> Instance with String IDs -> **`afterDecode` Hook** (ID-to-Index Resolution) -> Finalized PlayerConfig Instance -> Asset Registry Cache -> Game Systems (via PlayerEntity)


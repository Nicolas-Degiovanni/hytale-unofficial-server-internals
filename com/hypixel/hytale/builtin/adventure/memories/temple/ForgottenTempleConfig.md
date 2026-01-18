---
description: Architectural reference for ForgottenTempleConfig
---

# ForgottenTempleConfig

**Package:** com.hypixel.hytale.builtin.adventure.memories.temple
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ForgottenTempleConfig {
```

## Architecture & Concepts
ForgottenTempleConfig is a passive data model, not an active service. Its primary architectural role is to serve as a strongly-typed, in-memory representation of configuration data loaded from an external asset file, likely JSON. This class encapsulates settings specific to the "Forgotten Temple" adventure zone, decoupling the zone's game logic from the raw data format on disk.

The most critical architectural component is the static **CODEC** field. This `BuilderCodec` instance defines the serialization and deserialization contract for the class. The Hytale engine's asset loading system uses this codec to automatically parse a configuration file and construct a fully populated ForgottenTempleConfig object. This pattern centralizes the data mapping logic within the data model itself, ensuring consistency and removing the need for external parsing factories.

The class also demonstrates a common engine optimization pattern in the `getRespawnSoundIndex` method. It translates a human-readable string identifier, `respawnSound`, into a more performant integer index used by the core `SoundEvent` system. This avoids repeated, costly string lookups in performance-critical game loops.

## Lifecycle & Ownership
- **Creation:** An instance of ForgottenTempleConfig is created exclusively by the Hytale `Codec` deserialization pipeline when the engine loads the corresponding adventure zone assets. It is never instantiated directly with the `new` keyword in game logic.
- **Scope:** The object's lifetime is bound to the adventure zone it configures. It is typically held as a field within a higher-level zone manager or world context object and persists as long as that zone is active in memory.
- **Destruction:** The object is marked for garbage collection when the parent adventure zone is unloaded from the game world, for example, when a player leaves the area or the server shuts down.

## Internal State & Concurrency
- **State:** The internal state is **Mutable** only during the deserialization process managed by the `CODEC`. After instantiation, it should be treated as an **Effectively Immutable** object. The game logic reads from this configuration but must not attempt to modify it at runtime.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created and populated on a background asset-loading thread and subsequently read by the main game thread. Any cross-thread access must be externally synchronized. Unsynchronized access, especially during the initial load, will lead to race conditions and undefined behavior.

## API Surface
The public API is designed for read-only access to the configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMinYRespawn() | double | O(1) | Retrieves the world Y-coordinate below which a player is considered to have fallen out of the world and must be respawned. |
| getRespawnSound() | String | O(1) | Retrieves the raw string identifier for the sound event to be played upon player respawn. Returns null if not defined. |
| getRespawnSoundIndex() | int | O(1) avg | Translates the respawn sound ID into a numerical index via the global SoundEvent asset map. This is the preferred method for triggering the sound. |

## Integration Patterns

### Standard Usage
Game systems should retrieve the configuration object from a central context or registry associated with the game world or specific zone. The values are then used to drive game mechanics.

```java
// Example from a hypothetical PlayerRespawnSystem
void handlePlayerFall(Player player, WorldZone zone) {
    ForgottenTempleConfig config = zone.getConfig(ForgottenTempleConfig.class);
    
    if (player.getPosition().getY() < config.getMinYRespawn()) {
        int soundIndex = config.getRespawnSoundIndex();
        SoundSystem.playSound(soundIndex, player.getPosition());
        player.teleportTo(zone.getSpawnPoint());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ForgottenTempleConfig()`. This bypasses the asset loading system and results in an object with default, unconfigured values, leading to incorrect game behavior.
- **Runtime Modification:** Do not use reflection or other means to modify the state of this object after it has been loaded. Configuration is expected to be static for the duration of the object's lifecycle.

## Data Pipeline
The data for this object originates from a source asset file and is processed by the engine's serialization systems before being consumed by game logic.

> Flow:
> AdventureZone.json (Asset File) -> Asset Loading Service -> `Codec` Deserializer -> **ForgottenTempleConfig** (In-Memory Instance) -> Game Systems (e.g., PlayerStateSystem)


---
description: Architectural reference for DamageEffects
---

# DamageEffects

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server.combat
**Type:** Configuration Object

## Definition
```java
// Signature
public class DamageEffects implements NetworkSerializable<com.hypixel.hytale.protocol.DamageEffects> {
```

## Architecture & Concepts

The DamageEffects class is a data-driven configuration object that encapsulates the sensory feedback for a damage event. It is not a service or a manager; rather, it serves as an immutable template that describes *what should happen* when damage is applied, such as which particles to spawn, sounds to play, and camera effects to trigger.

This class acts as a critical bridge between abstract game logic (e.g., a sword strike) and the concrete, low-level rendering and audio systems. It is designed to be defined in external asset files (likely JSON or HOCON) and is deserialized into a Java object at server startup via its static **CODEC** field.

A key architectural feature is its two-phase initialization. During asset loading, string-based identifiers for assets (e.g., a sound event ID like *hytale:weapon.sword.hit_flesh*) are deserialized. Immediately after, the **processConfig** method is invoked, which resolves these human-readable strings into high-performance integer indices. These indices are then used in performance-critical game loops, avoiding costly string comparisons or hash map lookups during combat calculations.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the engine's asset loading system using the provided static **BuilderCodec**. This process occurs once during the server's bootstrap phase when game configuration files are parsed. Direct instantiation via the *new* keyword is an anti-pattern and will result in a non-functional object.
-   **Scope:** An instance of DamageEffects persists for the entire server session. It is loaded once and cached in memory, typically as a field within a larger configuration object like a weapon or spell definition. It is treated as a shared, read-only resource.
-   **Destruction:** The object is de-referenced and garbage collected only when the server shuts down or a hot-reload of assets occurs.

## Internal State & Concurrency

-   **State:** The object is stateful but becomes effectively immutable after its initial creation and processing. The codec sets its primary fields, and the internal **processConfig** method mutates the transient *index* fields (e.g., worldSoundEventIndex). This mutation happens only once in a controlled, single-threaded context during server startup.
-   **Thread Safety:** The class is thread-safe for all read operations. Because its state does not change after initialization, multiple game threads (e.g., entity processing threads) can safely access a shared instance without requiring locks or other synchronization primitives. The methods that apply effects, such as **spawnAtEntity**, are designed to be called from within a thread-safe context like an Entity Component System (ECS) CommandBuffer.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addToDamage(Damage damageEvent) | void | O(1) | Populates a **Damage** event object with metadata (sound indices, particle definitions) for deferred processing by the core damage system. |
| spawnAtEntity(CommandBuffer, Ref) | void | O(N) | Immediately triggers the configured effects in the world at a specific entity's location. Complexity is O(N) where N is the number of players within the effect's view distance. |
| toPacket() | com.hypixel.hytale.protocol.DamageEffects | O(M) | Serializes the object into a network-optimized packet for the client. Complexity is O(M) where M is the number of particles to serialize. |

## Integration Patterns

### Standard Usage

DamageEffects is not used directly but is retrieved from a higher-level configuration object. Its methods are then used to either describe effects for a later system or to spawn them immediately.

```java
// Example: Applying effects as part of a damage calculation
// Assume 'weaponConfig' is loaded from an asset file and contains a DamageEffects field.

Damage damageEvent = new Damage(...);
WeaponConfig weaponConfig = getWeaponConfigFor("iron_sword");

// Retrieve the pre-loaded DamageEffects and apply its metadata to the event
weaponConfig.getDamageEffects().addToDamage(damageEvent);

// The damageEvent is now ready to be processed by the damage system
damageSystem.apply(damageEvent);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new DamageEffects()`. This bypasses the codec and the critical **processConfig** step, leaving all asset indices uninitialized (typically as 0 or MIN_VALUE). The object will fail to produce any effects.
-   **Runtime Modification:** Do not modify the fields of a DamageEffects object after it has been loaded. These objects are shared across the server. Modifying one instance will affect all logic that uses it, leading to unpredictable behavior.
-   **Incorrect Context:** Calling **spawnAtEntity** outside of the server's main entity update loop (i.e., without a valid CommandBuffer) will lead to exceptions or race conditions. This method is designed to work within the ECS transaction model.

## Data Pipeline

The data for a DamageEffects object follows a clear path from configuration file to in-game effect.

> **Loading-Time Flow:**
> Game Asset File (JSON/HOCON) -> Server Asset Loader -> **BuilderCodec** -> **DamageEffects Instance** (in memory with resolved integer indices)

> **Game-Time Flow (Deferred):**
> Combat Logic -> Retrieves **DamageEffects** -> `addToDamage(event)` -> Damage System -> Spawns Effects Based on Event Metadata

> **Game-Time Flow (Immediate):**
> Combat Logic -> Retrieves **DamageEffects** -> `spawnAtEntity(commandBuffer, ref)` -> CommandBuffer Execution -> Effects Spawned in World


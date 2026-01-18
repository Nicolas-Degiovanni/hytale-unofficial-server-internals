---
description: Architectural reference for AbilityEffects
---

# AbilityEffects

**Package:** com.hypixel.hytale.server.core.asset.type.entityeffect.config
**Type:** Data Model

## Definition
```java
// Signature
public class AbilityEffects implements NetworkSerializable<com.hypixel.hytale.protocol.AbilityEffects> {
```

## Architecture & Concepts
The AbilityEffects class is a server-side data model that represents a specific piece of an entity effect's configuration. Its sole responsibility is to define a set of player or entity interactions that should be disabled while the parent entity effect is active.

This class is not a service or a manager; it is a passive data container. Its structure and lifecycle are dictated entirely by the Hytale codec system, as evidenced by the static final **CODEC** field. This `BuilderCodec` is responsible for deserializing configuration data from on-disk asset files (e.g., JSON or HOCON) into a hydrated in-memory instance of this class.

Architecturally, AbilityEffects serves as a typed, in-memory representation of static game design data. It is a critical component in the data pipeline that flows from game assets to the server's core logic, specifically the InteractionModule, which consumes this data to enforce gameplay rules. The implementation of the NetworkSerializable interface signifies its role in state synchronization, allowing the server to inform clients of these interaction restrictions.

## Lifecycle & Ownership
- **Creation:** Instances of AbilityEffects are created exclusively by the `hytale-codec` framework during the server's asset loading phase. The protected no-argument constructor exists solely for use by the reflection-based `BuilderCodec`. Manual instantiation is a design violation.
- **Scope:** The lifetime of an AbilityEffects object is strictly bound to the lifecycle of its parent entity effect asset. It persists in memory as long as the asset is loaded and is treated as immutable configuration data.
- **Destruction:** The object is eligible for garbage collection when the server unloads the corresponding asset bundle or shuts down. There is no manual destruction or cleanup mechanism.

## Internal State & Concurrency
- **State:** The internal state is effectively **immutable**. The primary constructor performs a defensive copy of the incoming `Set` into an `EnumSet`, preventing any external modifications to the collection after instantiation. The state consists of a single `Set` of InteractionType enums.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature, instances can be safely read by multiple threads simultaneously without locks or other synchronization primitives. This is critical, as game logic running on different threads (e.g., player update ticks) may need to query this configuration concurrently.

## API Surface
The public API is minimal, reflecting its role as a simple data holder.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.AbilityEffects | O(N) | Serializes the internal state into a network packet. N is the number of disabled interactions. This is the primary mechanism for communicating the effect's restrictions to the client. |

## Integration Patterns

### Standard Usage
Developers should never create instances of this class directly. Instead, they should retrieve it from a parent configuration object that has been loaded by the asset system.

```java
// Correctly accessing the configuration from a parent asset
EntityEffectConfig effectConfig = assetManager.get("my_mod:my_poison_effect");
AbilityEffects abilityEffects = effectConfig.getAbilityEffects();

// Game logic then consumes the data
if (abilityEffects != null) {
    // The InteractionModule would use the contained set to validate player actions
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AbilityEffects()`. All configuration must be defined in asset files to be loaded by the codec system. Bypassing this breaks the asset pipeline and can lead to inconsistent game state.
- **State Mutation:** Do not attempt to use reflection or other means to modify the internal `disabled` set after the object has been created. The system relies on this configuration being immutable.

## Data Pipeline
AbilityEffects sits in the middle of the data flow from static configuration files to the client's game state.

> Flow:
> Asset File (JSON/HOCON) -> `BuilderCodec` Deserializer -> **AbilityEffects Instance** -> InteractionModule (Server Logic) -> `toPacket()` -> Network Packet -> Client Game State


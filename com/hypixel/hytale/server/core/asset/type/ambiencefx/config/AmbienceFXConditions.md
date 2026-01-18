---
description: Architectural reference for AmbienceFXConditions
---

# AmbienceFXConditions

**Package:** com.hypixel.hytale.server.core.asset.type.ambiencefx.config
**Type:** Transient Data Model

## Definition
```java
// Signature
public class AmbienceFXConditions implements NetworkSerializable<com.hypixel.hytale.protocol.AmbienceFXConditions> {
```

## Architecture & Concepts
AmbienceFXConditions is a server-side data model that encapsulates the complete set of rules for when an ambient effect should be active in the game world. It serves as the critical link between human-readable asset configuration files (e.g., JSON) and the server's high-performance runtime evaluation engine.

The class is fundamentally defined by its static **CODEC** field. This `BuilderCodec` is responsible for deserializing the asset configuration from disk into a hydrated Java object. This pattern centralizes all loading, validation, and inheritance logic, ensuring consistency across all ambient effects.

A key architectural function of this class is its implementation of the NetworkSerializable interface. The `toPacket` method translates this configuration object into a network-optimized protocol buffer message. During this translation, it performs a vital optimization: string-based identifiers (like `environmentIds`) and tag patterns are resolved into more efficient integer indices. This significantly reduces network packet size and the computational load on the client, which can work with simple integer comparisons instead of string operations.

This object represents a single node in the server's environmental decision tree. The Ambience System evaluates these conditions against a player's context (location, time, weather) to determine the appropriate soundscape and visual effects.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's asset loading system via the static **CODEC**. The protected constructor prevents direct instantiation. An object is created for each AmbienceFX asset defined in the game's configuration files.
- **Scope:** The lifetime of an AmbienceFXConditions object is bound to the server's asset registry. It is loaded on server startup and persists in memory for the entire server session.
- **Destruction:** The object is marked for garbage collection when the server shuts down or when a full asset reload discards the existing asset registry. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The object is highly mutable during the asset loading phase. The **CODEC** populates its fields from the configuration file, and the `afterDecode` hook triggers the `processConfig` method. This finalizes the internal state by resolving string IDs into integer indices. After this loading and processing phase, the object should be treated as **effectively immutable**.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed on the server's main thread during the asset loading bootstrap sequence. Subsequent reads from game logic threads are safe, but any concurrent modification would break the internal consistency between ID fields (e.g., `weatherIds`) and their corresponding transient index fields (e.g., `weatherIndices`), leading to network corruption or runtime exceptions.

## API Surface
The public API is primarily composed of simple getters. The most significant method is `toPacket`, which is part of its contract as a NetworkSerializable object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.AmbienceFXConditions | O(N) | Serializes the object into its network protocol equivalent. Resolves all string IDs and tags to integer indices. |
| isNever() | boolean | O(1) | Returns true if the effect is permanently disabled by configuration. |
| getEnvironmentIds() | String[] | O(1) | Returns the raw string identifiers for required environments. |
| getEnvironmentIndices() | int[] | O(1) | Returns the resolved integer indices for required environments. |
| getAltitude() | Range | O(1) | Returns the valid altitude range for this effect. |
| getDayTime() | Rangef | O(1) | Returns the valid time-of-day range for this effect. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay or mod developers. It is an internal component managed by the server's Ambience System. The engine retrieves these condition objects from loaded assets and passes them to an evaluation service.

```java
// Hypothetical engine-level usage by an AmbienceEvaluationSystem

// 1. Retrieve the asset from the registry
AmbienceFX ambienceAsset = assetRegistry.get("hytale:forest_night_crickets");
AmbienceFXConditions conditions = ambienceAsset.getConditions();

// 2. The system evaluates these conditions against a player's state
PlayerState playerState = player.getCurrentState();
if (matches(playerState, conditions)) {
    // 3. If matched, activate the effect for the player
    ambienceService.playEffect(player, ambienceAsset);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new AmbienceFXConditions()`. The object will be incomplete and lack critical data provided by the asset loader. All instances must originate from the **CODEC**.
- **Post-Load Modification:** Do not modify the state of a loaded AmbienceFXConditions object. Changing a field like `weatherIds` after the `processConfig` method has run will cause a fatal desynchronization with the `weatherIndices` field, resulting in incorrect client behavior or network errors.
- **Manual Serialization:** Do not attempt to serialize this class for network transmission using a generic serializer. The `toPacket` method performs essential, non-obvious data transformations (ID-to-index mapping) that are required for client compatibility.

## Data Pipeline
The data for this object follows a clear, one-way pipeline from configuration on disk to a network-optimized message sent to the client.

> Flow:
> JSON Asset File -> Engine's BuilderCodec -> **AmbienceFXConditions** (In-Memory Object) -> `processConfig()` Hydration -> `toPacket()` Serialization -> Network Packet -> Game Client


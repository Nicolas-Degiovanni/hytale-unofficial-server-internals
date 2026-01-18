---
description: Architectural reference for VoidEventConfig
---

# VoidEventConfig

**Package:** com.hypixel.hytale.builtin.portals.components.voidevent.config
**Type:** Configuration Model / Transient

## Definition
```java
// Signature
public class VoidEventConfig {
```

## Architecture & Concepts
The VoidEventConfig class is a passive data structure that serves as a blueprint for a "Void Event" within the game engine. It is not an active system or manager; rather, it is a configuration object deserialized from game assets, likely JSON files.

Its primary role is to encapsulate all tunable parameters for a void event—such as duration, stage progression, and associated visual and audio effects. The engine's event management systems consume instances of this class to orchestrate the event's behavior at runtime.

The class leverages Hytale's powerful **Codec** system for its construction. The static **CODEC** field defines the schema for deserialization, mapping keys from an asset file directly to the fields of the class. This pattern decouples the game logic from the raw configuration data, allowing designers to modify event behavior without changing engine code.

## Lifecycle & Ownership
- **Creation:** Instances are exclusively created by the Hytale **Codec** system during asset loading. The static **CODEC** field acts as the factory. Direct instantiation using the constructor is an invalid and unsupported operation. A critical post-processing step, `processConfig`, is invoked automatically by the codec via the `afterDecode` hook. This step sorts the event stages, making the object ready for consumption.

- **Scope:** The lifetime of a VoidEventConfig instance is tied to the asset it was loaded from. It persists in memory as long as that asset is required by the current game state (e.g., a specific world or zone) and is eligible for garbage collection once the asset is unloaded.

- **Destruction:** There is no explicit destruction method. The Java Garbage Collector reclaims the memory once all references to the instance are released.

## Internal State & Concurrency
- **State:** The object is mutable only during the deserialization and `afterDecode` processing phase. After this initial construction, its state is effectively immutable. All public methods are getters, providing read-only access to the configuration data. The `stagesSortedByStartTime` list is a derivative cache, populated once upon creation to optimize runtime access.

- **Thread Safety:** This class is **not thread-safe** during its construction phase. However, once fully materialized by the Codec system, the resulting instance is safe to be read by multiple threads concurrently. All subsequent access is read-only, and no internal state is modified.

    **Warning:** Do not share an instance of this class between threads before the Codec system has fully completed the `afterDecode` process.

## API Surface
The public API is designed for read-only access to the event's configured parameters.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDurationSeconds() | int | O(1) | Returns the total configured duration of the event in seconds. |
| getShouldStartAfterSeconds(int) | int | O(1) | Calculates the event's start time relative to a portal's time limit. |
| getInvasionPortalConfig() | InvasionPortalConfig | O(1) | Returns the nested configuration for enemy invasion portals. |
| getStages() | VoidEventStage[] | O(1) | Returns the raw, unsorted array of event stages as defined in the asset. |
| getStagesSortedByStartTime() | List<VoidEventStage> | O(1) | Returns a pre-sorted list of event stages, optimized for sequential processing. |
| getMusicAmbienceFX() | String | O(1) | Returns the asset ID for the music to be played during the event. May be null. |

## Integration Patterns

### Standard Usage
A game system, such as an event or zone manager, should retrieve a fully-formed instance of this class from the asset loading pipeline. The system then reads its properties to drive game logic.

```java
// Example: A ZoneManager retrieving the event configuration for a portal
// Note: AssetService is a hypothetical class for demonstration.

AssetService assetSvc = context.getService(AssetService.class);
VoidEventConfig eventConfig = assetSvc.load("myworld.void_event_config");

// Use the config to schedule game events
int duration = eventConfig.getDurationSeconds();
for (VoidEventStage stage : eventConfig.getStagesSortedByStartTime()) {
    scheduler.scheduleTask(stage.getSecondsInto(), () -> triggerStage(stage));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new VoidEventConfig()`. This bypasses the Codec deserialization and the critical `processConfig` method, resulting in a partially initialized object with a null `stagesSortedByStartTime` list. This will lead to NullPointerExceptions at runtime.

- **State Mutation:** The collections returned by `getStages` and `getStagesSortedByStartTime` must be treated as read-only. Modifying them externally will corrupt the object's state and lead to unpredictable behavior in the systems that consume this configuration.

## Data Pipeline
The data for a VoidEventConfig instance originates from a static asset file and is transformed into an in-memory object by the engine's core systems.

> Flow:
> Game Asset File (.json) -> Hytale Asset Loader -> **BuilderCodec** -> **VoidEventConfig Instance** -> Game Event System


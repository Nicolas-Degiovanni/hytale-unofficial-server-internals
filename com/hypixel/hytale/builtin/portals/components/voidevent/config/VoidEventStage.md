---
description: Architectural reference for VoidEventStage
---

# VoidEventStage

**Package:** com.hypixel.hytale.builtin.portals.components.voidevent.config
**Type:** Configuration Model / DTO

## Definition
```java
// Signature
public class VoidEventStage {
```

## Architecture & Concepts
The VoidEventStage class is a passive data model that represents a single, time-stamped phase within a larger in-game "Void Event". It does not contain any logic itself; its sole purpose is to hold configuration data deserialized from game asset files.

Architecturally, this class serves as a strongly-typed schema for a portion of a configuration file. The static **CODEC** field is the most critical component, acting as the bridge between the raw data on disk (e.g., JSON or HOCON) and this in-memory Java representation. It leverages Hytale's core serialization system to declaratively map asset keys like *SecondsInto* and *ForcedWeather* to the class's private fields.

Instances of VoidEventStage are almost always managed in collections by a parent configuration object, which is then consumed by a higher-level manager system responsible for orchestrating the event's progression over time.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale serialization engine via the static **CODEC** field. The engine invokes the private constructor during the asset loading process when it parses a corresponding data structure in a configuration file. Manual instantiation is an anti-pattern.
- **Scope:** The lifetime of a VoidEventStage object is bound to its parent configuration object. It persists in memory as long as the parent asset is loaded, typically for the duration of a server session or until a hot-reload of assets is triggered.
- **Destruction:** The object is eligible for garbage collection once its parent configuration object is unloaded and no longer referenced. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The state of a VoidEventStage is **effectively immutable**. While the fields are not declared as final, the design pattern of creation-via-codec implies that the object's state is set once upon deserialization and is not intended to be modified thereafter. It serves as a read-only snapshot of the source configuration.
- **Thread Safety:** The class is **conditionally thread-safe**. It is perfectly safe for concurrent reads from multiple threads, as its state is not expected to change after initialization. It is **not safe** for concurrent writes, but mutation is not an intended use case. The static **CODEC** itself is thread-safe and can be used to deserialize stages from multiple threads.

## API Surface
The public API is minimal, exposing only read-only access to the underlying configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSecondsInto() | int | O(1) | Returns the time offset in seconds from the start of the event when this stage becomes active. |
| getForcedWeatherId() | String | O(1) | Returns the asset identifier for the weather that should be active during this stage. May be null. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with a lone VoidEventStage. Instead, they will access a list of stages from a parent configuration object and use it to drive game logic.

```java
// Example: A hypothetical manager processing event stages
VoidEventConfig eventConfig = assetManager.load("my_mod:my_void_event");

for (VoidEventStage stage : eventConfig.getStages()) {
    if (gameTime >= stage.getSecondsInto()) {
        // Apply weather, trigger effects, etc.
        world.setWeather(stage.getForcedWeatherId());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new VoidEventStage()`. The object's state would be uninitialized, and it bypasses the entire configuration-as-code paradigm. The only valid creator is the Hytale codec system.
- **Post-Load Mutation:** Do not modify the state of a VoidEventStage instance after it has been loaded. Other systems rely on this data being a faithful representation of the on-disk asset. Modifying it at runtime can lead to desynchronization and unpredictable behavior.

## Data Pipeline
This class is a destination for data, not a processor. The pipeline describes how raw configuration data is transformed into a VoidEventStage instance.

> Flow:
> Asset File (e.g., JSON) -> AssetManager -> **BuilderCodec<VoidEventStage>** -> In-Memory **VoidEventStage** Instance -> Event Management System


---
description: Architectural reference for BuilderSensorEntityPrioritiserAttitude
---

# BuilderSensorEntityPrioritiserAttitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.prioritisers.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorEntityPrioritiserAttitude extends BuilderSensorEntityPrioritiserBase {
```

## Architecture & Concepts
The BuilderSensorEntityPrioritiserAttitude is a configuration-time component within the Hytale NPC AI asset pipeline. Its sole responsibility is to parse a specific JSON configuration block and construct an instance of SensorEntityPrioritiserAttitude. It acts as a translator, converting a declarative JSON definition into a functional, runtime-ready AI component.

This class is part of a larger **Builder Pattern** implementation used throughout the server to construct complex objects from asset files. It is discovered and instantiated by the asset loading system, which then injects the relevant JSON data into it.

The component it builds, SensorEntityPrioritiserAttitude, is used by an NPC's AI "sensor" systems. These sensors detect nearby entities, and the prioritiser provides the logic to rank those entities based on their "Attitude" (e.g., Hostile, Friendly, Neutral). This allows an NPC to decide which targets to focus on, ignore, or flee from. This builder specifically configures the *order of importance* for those attitudes.

### Lifecycle & Ownership
-   **Creation:** Instantiated reflectively by the server's NPC asset loading system when it encounters the corresponding component type in an NPC's JSON definition file. It is never created directly by game logic code.
-   **Scope:** The object's lifetime is extremely short and confined to the asset loading process. It exists only to parse a single JSON element, hold the resulting state temporarily, and be consumed by a single call to its build method.
-   **Destruction:** After the `build` method is called and the final SensorEntityPrioritiserAttitude object is returned, the builder instance is no longer referenced by the asset loader and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** This class is mutable. Its primary internal state is the `prioritisedAttitudes` field, an EnumArrayHolder that is populated by the `readConfig` method. This state is transient and only meaningful during the brief period between parsing and building.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. The server's asset loading pipeline guarantees that each builder instance is used in a single-threaded, synchronous manner for the duration of its lifecycle.

## API Surface
The public API is designed for consumption by the automated asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorEntityPrioritiserAttitude | O(N) | Constructs the final runtime prioritiser object. N is the number of attitudes. |
| readConfig(JsonElement) | BuilderSensorEntityPrioritiserAttitude | O(N) | Parses the JSON data and populates the internal state. N is the number of attitudes in the JSON array. |
| getPrioritisedAttitudes(BuilderSupport) | Attitude[] | O(1) | Retrieves the parsed and ordered list of attitudes. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class via Java code. Instead, they define the component's properties within an NPC's JSON asset file. The engine uses the builder to realize this configuration.

A conceptual example of how the *engine* uses this class:

```java
// Engine-level code (conceptual)
// The asset loader finds a builder for the component type "Attitude"
BuilderSensorEntityPrioritiserAttitude builder = new BuilderSensorEntityPrioritiserAttitude();

// It passes the relevant JSON data to the builder
JsonElement configData = getJsonForComponent("Attitude");
builder.readConfig(configData);

// It then builds the final runtime object
BuilderSupport support = createBuilderSupport();
SensorEntityPrioritiserAttitude prioritiser = builder.build(support);

// The 'prioritiser' is then attached to the NPC's AI sensor
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderSensorEntityPrioritiserAttitude()` in game logic. The asset system manages the lifecycle.
-   **State Manipulation:** Do not attempt to modify the builder's state after `readConfig` has been called. The internal state is considered sealed after parsing.
-   **Calling Build Before Read:** Invoking `build` before `readConfig` will result in an unconfigured or default-configured prioritiser, leading to incorrect AI behavior.
-   **Instance Reuse:** Builder instances are single-use. Do not cache and reuse a builder to construct multiple prioritiser objects.

## Data Pipeline
This builder is a key step in the transformation of static data into a live game object.

> Flow:
> NPC JSON Asset File -> GSON Parser -> `JsonElement` -> **BuilderSensorEntityPrioritiserAttitude.readConfig** -> Internal State (EnumArrayHolder) -> **BuilderSensorEntityPrioritiserAttitude.build** -> SensorEntityPrioritiserAttitude Instance -> NPC AI Sensor Component


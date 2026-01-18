---
description: Architectural reference for BuilderEntityFilterAltitude
---

# BuilderEntityFilterAltitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterAltitude extends BuilderEntityFilterBase {
```

## Architecture & Concepts

The BuilderEntityFilterAltitude class is a factory component within the server's NPC asset definition pipeline. It is not a runtime game component; rather, its sole purpose is to translate a declarative JSON configuration into a functional IEntityFilter object, specifically an EntityFilterAltitude instance.

This class embodies the Builder pattern, cleanly separating the complex, stateful process of configuration parsing and validation from the final, immutable filter object used by the NPC logic. It acts as a deserializer and constructor, taking raw JSON data as input and producing a validated, ready-to-use game logic component.

The use of the internal NumberArrayHolder field is a critical design choice. It signifies a deferred resolution mechanism, where the final double array value for the altitude range is not resolved until the build method is invoked. This allows the system to potentially inject dynamic, context-aware values during the final construction phase, managed by the BuilderSupport context object.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderEntityFilterAltitude is created dynamically by the NPC asset loading system whenever it encounters a filter definition of the corresponding type within an NPC's behavior JSON file. It is never created directly by game logic developers.
-   **Scope:** The object's lifetime is exceptionally brief and confined to the asset loading phase. It is created, configured once via readConfig, used once to produce an object via build, and is then immediately eligible for garbage collection.
-   **Destruction:** The object is a short-lived, temporary container for configuration data. It is implicitly destroyed by the garbage collector after the build method completes and its result has been passed to the parent system. No manual cleanup is necessary.

## Internal State & Concurrency

-   **State:** This class is stateful. The internal altitudeRange field holds the parsed configuration data after readConfig is called. This state is essential for the subsequent call to build. The object is designed for a single, linear sequence of operations: create, configure, build.

-   **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to operate exclusively within a single-threaded asset loading context. Accessing an instance from multiple threads will lead to race conditions and unpredictable behavior, as the internal state is mutable and unsynchronized. The broader asset pipeline is responsible for ensuring thread containment.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns the final EntityFilterAltitude instance. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses and validates the JSON configuration. N is the number of elements in the array. Throws exceptions on invalid data. |
| getAltitudeRange(BuilderSupport) | double[] | O(1) | Resolves the configured altitude range using the provided context. |

## Integration Patterns

### Standard Usage

The standard interaction with this class is managed entirely by the server's asset loading framework. A developer defines the filter in a JSON file, and the system handles the builder's lifecycle automatically.

```java
// This lifecycle is managed by the NPC asset pipeline.

// 1. A JSON definition for the filter is located in an asset file.
JsonElement filterConfig = getFilterConfigFromJson(); // e.g., { "AltitudeRange": [10.0, 50.0] }

// 2. The system instantiates the corresponding builder.
BuilderEntityFilterAltitude builder = new BuilderEntityFilterAltitude();

// 3. The configuration is read, validated, and stored internally.
builder.readConfig(filterConfig);

// 4. The final filter object is constructed using runtime context.
BuilderSupport support = getBuilderSupportFromContext();
IEntityFilter filter = builder.build(support);

// The 'filter' instance is now attached to an NPC's behavior tree.
// The 'builder' instance is now discarded.
```

### Anti-Patterns (Do NOT do this)

-   **State Re-use:** Do not attempt to reuse a BuilderEntityFilterAltitude instance. Each instance is designed for a single `readConfig` -> `build` operation. Reusing an instance can lead to state corruption and unpredictable filter behavior.
-   **Build Before Configure:** Calling build before readConfig has been successfully invoked will result in an unconfigured or default-state filter, which will likely fail validation or produce incorrect behavior at runtime.
-   **Direct Instantiation:** Game logic developers should never instantiate this class using `new`. Filter creation should always be done declaratively through JSON asset files.

## Data Pipeline

The primary flow for this component is one of configuration transformation, not real-time game data processing.

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> JsonElement -> **BuilderEntityFilterAltitude.readConfig()** -> Internal State (NumberArrayHolder) -> **BuilderEntityFilterAltitude.build()** -> EntityFilterAltitude Instance -> NPC Targeting System


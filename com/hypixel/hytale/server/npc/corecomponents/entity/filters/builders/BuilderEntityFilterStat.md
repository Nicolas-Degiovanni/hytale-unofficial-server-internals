---
description: Architectural reference for BuilderEntityFilterStat
---

# BuilderEntityFilterStat

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterStat extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterStat class is a factory and deserializer responsible for constructing an EntityFilterStat instance from a JSON configuration. It is a critical component of the server's NPC behavior system, which allows game designers to define complex entity logic declaratively.

This builder's primary role is to translate a high-level JSON definition into a concrete, runtime-ready filter object. This filter is then used within an NPC's behavior tree or state machine to make decisions based on an entity's stats, such as health, mana, or energy. For example, a designer could use this filter to create a condition like "trigger this behavior only when the target's health is below 50% of its maximum health".

A key architectural pattern employed here is **deferred resolution**. The builder does not immediately resolve asset strings (like "health") into their internal integer IDs. Instead, it stores them in specialized container objects like AssetHolder and EnumHolder. The actual resolution is deferred until the final `build` method is called, at which point a BuilderSupport context is provided. This context contains the necessary runtime information (like the asset-to-ID mapping) to create the final, optimized EntityFilterStat object. This decouples the static asset definition from the live server environment.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderEntityFilterStat is created by the server's asset loading framework whenever it encounters a filter of this type within an NPC's JSON definition file. It is never instantiated directly by game logic code.
- **Scope:** The object's lifetime is extremely short and confined to the asset loading and compilation phase. It exists only to parse a JSON fragment and produce a single IEntityFilter instance.
- **Destruction:** Once the `build` method has been called and the resulting EntityFilterStat object has been returned, the builder instance is no longer referenced and becomes eligible for garbage collection. It holds no persistent state beyond this process.

## Internal State & Concurrency
- **State:** The internal state is highly mutable, but only during the configuration phase initiated by the `readConfig` method. The various Holder fields (stat, statTarget, etc.) are populated from the JSON input. After configuration, the state is effectively immutable from an external perspective.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and used within a single, synchronized asset-loading thread. Concurrent calls to `readConfig` or `build` will lead to unpredictable behavior and race conditions.

**WARNING:** Do not share instances of this builder across threads. The internal state is unprotected, and the deferred resolution mechanism relies on a thread-local or single-threaded context.

## API Surface
The public API is minimal, reflecting its specific role as a factory. The various `get...` methods are intended for internal use by the constructed EntityFilterStat object, not for external callers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns the configured EntityFilterStat instance. Throws exceptions if resolution fails. |
| readConfig(JsonElement) | Builder | O(k) | Populates the builder's internal state from a JSON object. *k* is the number of keys in the JSON. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the server's internal NPC asset pipeline. A developer will never interact with it directly. The framework performs the following sequence:

```java
// Conceptual example of framework usage
JsonElement filterConfig = parseNpcDefinitionFile(".../mob.json");

// 1. Framework instantiates the correct builder based on a "type" field in JSON
BuilderEntityFilterStat builder = new BuilderEntityFilterStat();

// 2. Framework configures the builder with the specific JSON data
builder.readConfig(filterConfig);

// 3. Framework provides a runtime context to build the final object
BuilderSupport support = assetLoadingContext.getBuilderSupport();
IEntityFilter finalFilter = builder.build(support);

// 4. The finalFilter is now integrated into an NPC's behavior tree
npcBehavior.addCondition(finalFilter);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderEntityFilterStat()` in game logic. The asset system manages the lifecycle.
- **Incomplete Configuration:** Do not call `build` before `readConfig` has been successfully invoked by the framework. The resulting filter will be in an invalid state and will cause runtime errors.
- **Reusing Builders:** Do not attempt to cache or reuse builder instances. They are cheap to create and are not designed to be reconfigured.

## Data Pipeline
The builder acts as a transformation step in the data pipeline that converts static configuration files into live game objects.

> Flow:
> NPC Definition (JSON File) -> Server Asset Loader -> **BuilderEntityFilterStat.readConfig()** -> **BuilderEntityFilterStat.build()** -> EntityFilterStat Instance -> NPC Behavior Tree


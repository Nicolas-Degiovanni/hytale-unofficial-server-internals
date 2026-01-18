---
description: Architectural reference for BuilderEntityFilterItemInHand
---

# BuilderEntityFilterItemInHand

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterItemInHand extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterItemInHand is a component within the server's NPC asset loading and behavior definition system. It implements the **Builder Pattern** to translate a declarative JSON configuration into a concrete, executable game logic object.

Its primary role is to act as a factory and deserializer for the EntityFilterItemInHand filter. During server startup or asset reloading, a central asset pipeline parses NPC definition files. When the pipeline encounters a filter of this specific type, it instantiates this builder and delegates the parsing of the corresponding JSON object to it.

This class is a plug-in to the broader `Builder` framework. It encapsulates the specific validation rules and data structures required to configure a filter that checks for items held by an entity. It separates the concern of *defining* a filter in a data format (JSON) from the concern of *implementing* the filter's runtime logic (the EntityFilterItemInHand class).

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level asset parsing system, likely a factory or service locator, when an "ItemInHand" entity filter is declared in an NPC's JSON definition. It is never created directly by game logic developers.
-   **Scope:** The lifecycle of a builder instance is extremely short. It exists only for the duration of parsing a single JSON configuration block.
-   **Destruction:** Once the `build` method is called and the final `EntityFilterItemInHand` object is returned to the asset system, the builder instance has served its purpose. It holds no further references and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** The internal state is **Mutable**. The `readConfig` method populates the `items` and `hand` fields based on the input JSON. These fields are specialized holders (`AssetArrayHolder`, `EnumHolder`) that store the configuration in an intermediate, unresolved state until the final `build` call.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context during the asset loading phase. Concurrent calls to `readConfig` would corrupt the internal state of the builder. The asset pipeline is responsible for ensuring that each builder instance is used by only one thread at a time.

## API Surface
The public API is designed for consumption by the asset loading framework, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | EntityFilterItemInHand | O(1) | Constructs the final, immutable filter object. This is the terminal operation for the builder. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses the JSON data, populates internal state, and performs validation. N is the size of the JSON element. |
| getItems(BuilderSupport) | String[] | O(1) | Retrieves the resolved item asset names. Requires a support context. |
| getHand(BuilderSupport) | WieldingHand | O(1) | Retrieves the resolved hand enum value. Requires a support context. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. The server's asset loading system orchestrates its use. The conceptual flow within the engine is as follows:

```java
// Conceptual example of engine-level usage
// Engine finds a filter definition in JSON and gets the corresponding builder.
BuilderEntityFilterItemInHand builder = assetSystem.getBuilderFor("EntityFilterItemInHand");

// Engine passes the specific JSON configuration to the builder.
JsonElement filterConfig = getFilterConfigFromJson(); // { "Items": ["minecraft:sword_*"], "Hand": "Main" }
builder.readConfig(filterConfig);

// Engine provides a context and builds the final object.
BuilderSupport support = assetSystem.createBuilderSupport();
IEntityFilter filter = builder.build(support);

// The resulting filter is then attached to an NPC behavior node.
behaviorNode.addFilter(filter);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderEntityFilterItemInHand()`. The asset system is responsible for managing the lifecycle of builders. Direct instantiation bypasses the framework and will fail.
-   **State Re-use:** Do not reuse a builder instance to parse multiple, different JSON configurations. Each builder is single-use and should be discarded after one `build` cycle.
-   **Premature Build:** Calling `build` before `readConfig` will result in a filter with default, likely incorrect, behavior.

## Data Pipeline
This builder is a key stage in the NPC asset-to-engine-object data pipeline. It transforms static data into executable logic.

> Flow:
> NPC Definition (JSON File) -> Engine JSON Parser -> **BuilderEntityFilterItemInHand.readConfig()** -> Internal State (AssetHolders) -> **BuilderEntityFilterItemInHand.build()** -> EntityFilterItemInHand (Instance) -> NPC Behavior Tree


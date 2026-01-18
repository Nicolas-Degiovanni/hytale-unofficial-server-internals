---
description: Architectural reference for BuilderActionSpawn
---

# BuilderActionSpawn

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionSpawn extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSpawn class is a transient configuration object responsible for translating a declarative JSON definition into a concrete, executable ActionSpawn instance. It is a critical component of the server's NPC asset loading pipeline, acting as a strongly-typed factory and data validator for NPC spawning behavior.

Its primary architectural role is to decouple the static data representation (JSON) from the runtime game logic (the ActionSpawn object). It achieves this through two core patterns:

1.  **The Builder Pattern:** It accumulates configuration state via the `readConfig` method and produces a final, immutable-by-convention object through its terminal `build` method.
2.  **The Holder Pattern:** Instead of storing raw primitive values, it uses specialized container classes like StringHolder, FloatHolder, and NumberArrayHolder. These Holders encapsulate the configuration values, deferring their final resolution until an ExecutionContext is available. This allows for dynamic values and context-sensitive lookups during the build process.

This class is not an action executor. It is exclusively a data-transfer and validation object used during the server's asset initialization phase.

### Lifecycle & Ownership
-   **Creation:** An instance of BuilderActionSpawn is created by the NPC asset parsing system whenever it encounters an action of type "spawn" within an NPC's JSON definition file. It is never instantiated directly by gameplay code.
-   **Scope:** Extremely short-lived. Its lifecycle is confined to the asset parsing and object construction phase. It exists only to be configured by `readConfig` and then immediately used in a single call to `build`.
-   **Destruction:** The object is intended to be discarded and becomes eligible for garbage collection immediately after the `build` method returns the final ActionSpawn instance. It holds no persistent references and is not registered in any service locator.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable during the invocation of the `readConfig` method, as this is when the various Holder fields are populated from the source JSON. After this phase, the object should be treated as effectively immutable.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, configured, and used by a single thread within the asset loading pipeline. Concurrent calls to `readConfig` or modification of its internal Holder fields will result in a corrupted state and unpredictable behavior. Synchronization is not implemented and must be handled by the calling system.

## API Surface
The public API is minimal, reflecting its focused role as a builder. The various `get...` methods are public primarily for access by the ActionSpawn constructor and are not intended for general consumption.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionSpawn | O(1) | **Terminal Operation.** Constructs and returns the final ActionSpawn object using the currently configured state. |
| readConfig(JsonElement) | BuilderActionSpawn | O(N) | Populates the builder's internal state from a JSON source. N is the number of properties in the JSON. This method is the primary entry point for configuration. |

## Integration Patterns

### Standard Usage
The standard interaction is entirely managed by the server's internal asset loading framework. A developer defining NPC behavior in JSON will trigger this class's usage implicitly.

```java
// This code is conceptual and represents the server's internal process.
// Do not replicate this pattern in gameplay code.

JsonElement spawnActionJson = parseNpcDefinitionFile(".../mob.json");
BuilderActionSpawn builder = new BuilderActionSpawn();

// 1. Configure the builder from the data source
builder.readConfig(spawnActionJson);

// 2. Build the final, runtime action object
ActionSpawn runtimeAction = builder.build(builderSupportContext);

// 3. The builder is now discarded. The runtimeAction is used by the NPC.
npc.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of BuilderActionSpawn directly in gameplay systems. It is not a gameplay object; it is a configuration-time tool.
-   **State Re-use:** Do not attempt to reuse a builder instance by calling `readConfig` multiple times. These objects are designed for a single, linear build process. Create a new instance for each action you need to build.
-   **Building Before Configuration:** Calling `build` on a new instance before `readConfig` has been successfully invoked will produce an ActionSpawn with default or null values, leading to server errors or unexpected NPC behavior.
-   **Concurrent Access:** Do not share a builder instance across multiple threads. The internal state is not protected by locks.

## Data Pipeline
The BuilderActionSpawn serves as a validation and transformation step in the data pipeline that converts static asset files into live server objects.

> Flow:
> NPC Asset JSON File -> GSON Parser -> `JsonElement` -> **BuilderActionSpawn**.`readConfig()` -> **BuilderActionSpawn** (Internal State Populated) -> `build()` -> `ActionSpawn` (Runtime Object) -> NPC Behavior System


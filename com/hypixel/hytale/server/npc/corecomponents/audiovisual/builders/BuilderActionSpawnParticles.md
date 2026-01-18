---
description: Architectural reference for BuilderActionSpawnParticles
---

# BuilderActionSpawnParticles

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionSpawnParticles extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSpawnParticles class is a transient, single-purpose object that functions as a factory within the server-side NPC asset pipeline. It embodies the Builder pattern, designed specifically to translate a declarative JSON configuration into a concrete, executable `ActionSpawnParticles` game object.

Its primary role is to act as an intermediary during the deserialization of an NPC's behavior definition. It decouples the raw data format (JSON) from the runtime game logic, allowing the structure of the `ActionSpawnParticles` class to change without breaking asset compatibility.

A key architectural feature is its use of the AssetHolder for the `particleSystem` field. This defers the validation and resolution of the particle system asset name until the `build` method is invoked. This lazy resolution strategy is critical for performance and for ensuring that all necessary game contexts are available when asset lookups are performed.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading system when an action of this type is encountered within an NPC's JSON definition. It is never managed by a dependency injection container or service registry.
- **Scope:** The lifecycle of a BuilderActionSpawnParticles instance is extremely short. It exists only for the duration required to parse a single JSON object and construct one `ActionSpawnParticles` instance.
- **Destruction:** The builder object is immediately eligible for garbage collection after the `build` method returns. Ownership of the resulting `ActionSpawnParticles` object is transferred to the calling system, typically an NPC's behavior tree or action sequencer.

## Internal State & Concurrency
- **State:** The internal state of this class is highly mutable. The `readConfig` method directly mutates the `particleSystem`, `range`, and `offset` fields. This state is transient and represents the configuration for a single `ActionSpawnParticles` object.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use within the asset loading pipeline. Concurrent calls to `readConfig` or `build` will lead to state corruption and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionSpawnParticles | O(1) | Constructs and returns the final ActionSpawnParticles object. Throws if the provided context is invalid. |
| readConfig(JsonElement) | BuilderActionSpawnParticles | O(1) | Populates the builder's internal state from a JSON source. This method must be called before `build`. |
| getParticleSystem(BuilderSupport) | String | O(1) | Resolves and returns the particle system asset name using the provided execution context. |
| getOffset() | Vector3d | O(1) | Returns a **new** Vector3d instance representing the configured offset. Creates a defensive copy. |

## Integration Patterns

### Standard Usage
The builder is intended to be used in a strict sequence by the asset loading framework. The caller is responsible for providing the JSON data and the build-time context.

```java
// Conceptual example of the asset pipeline's usage
BuilderActionSpawnParticles builder = new BuilderActionSpawnParticles();

// 1. Configure the builder from the source data
builder.readConfig(npcActionJson);

// 2. Provide the build-time context and construct the final object
ActionSpawnParticles action = builder.build(builderSupportContext);

// The 'builder' instance is now discarded.
// The 'action' instance is added to the NPC's behavior.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Never re-use a builder instance to create a second action. Its internal state is not reset between `build` calls. A new builder must be instantiated for each JSON action block.
- **Premature Build:** Do not call `build` before `readConfig` has been successfully executed. This will result in an action configured with uninitialized or default data, leading to runtime errors or incorrect visual effects.
- **State Mutation After Read:** Do not manually modify the public fields of this class after calling `readConfig`. The builder's state is considered sealed after parsing is complete.

## Data Pipeline
The class functions as a specific step in the data transformation pipeline that converts static asset files into live server objects.

> Flow:
> NPC Definition (*.json file*) -> JSON Parsing Service -> **BuilderActionSpawnParticles.readConfig()** -> **BuilderActionSpawnParticles.build()** -> `ActionSpawnParticles` Instance -> NPC Behavior Tree


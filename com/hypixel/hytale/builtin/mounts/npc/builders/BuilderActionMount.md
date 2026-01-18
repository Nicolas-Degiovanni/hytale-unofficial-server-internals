---
description: Architectural reference for BuilderActionMount
---

# BuilderActionMount

**Package:** com.hypixel.hytale.builtin.mounts.npc.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionMount extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionMount class is a component of the server-side NPC asset definition system. It serves as a factory and deserializer for creating a specific runtime behavior: enabling a player to mount an NPC. Its primary role is to translate a static data definition from a JSON asset file into a live, executable `ActionMount` object that can be integrated into an NPC's behavior tree.

This class follows the Builder pattern. It is instantiated by a higher-level asset parser which, upon identifying a "mount" action in the NPC's JSON definition, delegates the configuration and construction process to this specialized builder.

A key architectural feature is its use of `FloatHolder` and `StringHolder` objects instead of raw primitive types. These holders act as containers for configuration values that may be resolved dynamically at runtime using an `ExecutionContext`. This design allows for highly flexible and data-driven NPC configurations where values can be derived from variables or other contextual information, rather than being strictly hardcoded in the asset file.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderActionMount is created by the server's NPC asset loading system when it parses an NPC definition file and encounters an action of this type. It is not intended for manual instantiation by developers.
- **Scope:** The object's lifetime is extremely short and tied directly to the asset parsing process. It exists only to configure and build a single `ActionMount` instance.
- **Destruction:** The builder is eligible for garbage collection immediately after the `build` method is called and the resulting `ActionMount` object is returned. It holds no persistent state and is not retained by any long-lived system.

## Internal State & Concurrency
- **State:** BuilderActionMount is a stateful, mutable object. Its internal fields (`anchorX`, `anchorY`, `anchorZ`, `movementConfig`) are populated by the `readConfig` method. This state represents the complete configuration for the `ActionMount` object it is responsible for creating.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context during the asset loading phase. The lifecycle methods—instantiation, `readConfig`, and `build`—must be called serially on any given instance. Concurrent access would lead to a corrupted internal state and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionMount | O(1) | Constructs and returns the final `ActionMount` object using the internal configuration. |
| readConfig(JsonElement) | Builder<Action> | O(1) | Deserializes the provided JSON data, populating the internal state holders. Throws exceptions on missing or malformed data. |
| getAnchorX(BuilderSupport) | float | O(1) | Resolves and returns the configured X-axis mount anchor position. |
| getAnchorY(BuilderSupport) | float | O(1) | Resolves and returns the configured Y-axis mount anchor position. |
| getAnchorZ(BuilderSupport) | float | O(1) | Resolves and returns the configured Z-axis mount anchor position. |
| getMovementConfig(BuilderSupport) | String | O(1) | Resolves and returns the name of the movement configuration asset to use while mounted. |

## Integration Patterns

### Standard Usage
The builder is used by the asset loading system to construct an action from a JSON definition. The typical flow is to instantiate, configure, and immediately build.

```java
// Hypothetical asset loader context
JsonElement mountActionJson = parseNpcDefinition(".../mob.json");
BuilderSupport support = createBuilderSupportForNpc(npc);

// 1. The loader creates the correct builder for the action type
BuilderActionMount builder = new BuilderActionMount();

// 2. The builder is configured from the JSON data
builder.readConfig(mountActionJson);

// 3. The final, immutable Action object is built
ActionMount mountAction = builder.build(support);

// 4. The action is added to the NPC's behavior
npc.addAction(mountAction);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not reuse a builder instance to create multiple `ActionMount` objects. The internal state is not reset after a call to `build`, which would result in multiple identical action configurations. A new builder must be created for each action defined in the asset.
- **Instantiation Without Configuration:** Do not instantiate this class and call `build` without first calling `readConfig`. This will produce an `ActionMount` with default, uninitialized values, leading to runtime errors or unpredictable NPC behavior.
- **Manual State Modification:** Do not attempt to modify the internal `FloatHolder` or `StringHolder` fields directly. The public API surface is the only supported way to interact with this class.

## Data Pipeline
The BuilderActionMount acts as a transformation step in the NPC asset loading pipeline, converting declarative JSON data into an executable game object.

> Flow:
> NPC Asset JSON File -> Server JSON Parser -> **BuilderActionMount.readConfig** -> **BuilderActionMount.build** -> ActionMount Instance -> NPC Behavior Controller


---
description: Architectural reference for BuilderActionApplyEntityEffect
---

# BuilderActionApplyEntityEffect

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionApplyEntityEffect extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionApplyEntityEffect class is a configuration-time factory within the server's declarative NPC behavior system. It is not a component of the live game loop. Instead, its sole purpose is to translate a static data definition, typically from a JSON file, into a concrete, executable runtime object: the ActionApplyEntityEffect.

This class embodies the **Builder Pattern**, acting as an intermediary between raw asset data and a usable game logic component. It is responsible for parsing, validating, and preparing the parameters required for an NPC to apply a status effect to itself or a target.

A critical architectural concept demonstrated here is the use of **Holder** objects, such as AssetHolder and BooleanHolder. These objects decouple the *definition* of an action from its *runtime resolution*. The builder stores unresolved references during the initial parsing phase. These references are only resolved into concrete values (like an integer asset ID) later, when the `build` method is invoked with a specific BuilderSupport context. This allows for flexible and context-aware NPC definitions that can be compiled efficiently.

### Lifecycle & Ownership
-   **Creation:** An instance of BuilderActionApplyEntityEffect is created by the server's asset loading framework whenever it encounters an action of this type within an NPC's behavior definition file. It is never instantiated directly by game logic.
-   **Scope:** The lifecycle of this object is extremely short and confined to the asset parsing and compilation process. It exists only long enough to be configured via `readConfig` and then used to produce a final runtime action via `build`.
-   **Destruction:** The builder instance is immediately eligible for garbage collection after the `build` method returns. The system does not retain the builder; it only stores the ActionApplyEntityEffect object that was produced.

## Internal State & Concurrency
-   **State:** The internal state is mutable. The `readConfig` method populates the `entityEffect` and `useTarget` fields based on the input JSON. This state is considered "write-once" during the configuration phase of its lifecycle.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, configured, and used by a single thread within the server's asset loading pipeline. Concurrent access during the `readConfig` phase will lead to a corrupted and unpredictable state. This is considered safe by design, as asset compilation is a serialized, single-threaded process.

## API Surface
The public API is designed for a sequential, single-pass configuration process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionApplyEntityEffect | O(1) | Constructs the final, immutable runtime action. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderActionApplyEntityEffect | O(N) | Parses the JSON definition, populating internal state and performing validation. Returns self for chaining. |
| getEntityEffect(BuilderSupport) | int | O(1) | Resolves the configured entity effect asset into its runtime integer ID using the provided context. |
| isUseTarget(BuilderSupport) | boolean | O(1) | Resolves the configured boolean flag indicating if the effect applies to the NPC's target or itself. |

## Integration Patterns

### Standard Usage
The following example illustrates the conceptual flow managed by the server's asset system. Developers do not typically interact with this class directly.

```java
// This entire process is managed internally by the NPC asset system.

// 1. A new builder is instantiated for a specific action definition.
BuilderActionApplyEntityEffect builder = new BuilderActionApplyEntityEffect();

// 2. The system provides the raw JSON configuration from an asset file.
JsonElement actionConfig = loadActionConfigFromJson("my_npc.json");
builder.readConfig(actionConfig);

// 3. At build time, the system provides a context for resolving references.
BuilderSupport supportContext = createSupportContextForNpc();

// 4. The final, runtime action is created and stored in the NPC's behavior tree.
ActionApplyEntityEffect runtimeAction = builder.build(supportContext);

// The 'builder' instance is now discarded and will be garbage collected.
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not attempt to reuse a builder instance to configure multiple, distinct actions. Each action defined in a JSON file must be processed by a new, dedicated builder instance to ensure state isolation.
-   **Manual Instantiation:** This class is designed to be managed exclusively by the asset loading framework. Manually creating and managing instances can bypass critical validation and context-aware resolution steps, leading to runtime errors.
-   **Premature Resolution:** Calling `getEntityEffect` or `isUseTarget` before `readConfig` has been successfully invoked will result in exceptions or incorrect data, as the internal Holders will not have been initialized.

## Data Pipeline
This component operates within a configuration and compilation pipeline, not a real-time game data pipeline. Its function is to transform static data into an executable object.

> Flow:
> NPC Definition (JSON File) -> Server Asset Parser -> **BuilderActionApplyEntityEffect** (`readConfig`) -> `build()` -> ActionApplyEntityEffect (Runtime Object) -> Compiled NPC Behavior Tree


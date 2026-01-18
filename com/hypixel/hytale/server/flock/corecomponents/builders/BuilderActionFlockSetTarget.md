---
description: Architectural reference for BuilderActionFlockSetTarget
---

# BuilderActionFlockSetTarget

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient Builder

## Definition
```java
// Signature
public class BuilderActionFlockSetTarget extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionFlockSetTarget is a configuration-time object that acts as a factory for the runtime ActionFlockSetTarget. It is a critical component in the server's NPC behavior asset pipeline, responsible for translating a JSON definition into an executable game logic component.

Its primary role is to exist temporarily during asset deserialization. It reads a specific JSON block, validates the configuration, and stores the parsed parameters. Once configured, its build method is invoked to produce an immutable ActionFlockSetTarget instance, which is then integrated into an NPC's behavior tree or state machine.

This class embodies the Builder pattern, separating the complex construction of a game action from its final representation. This allows for a clean, declarative approach to defining NPC behavior in external JSON files while keeping the runtime action objects lightweight and immutable. A key feature is its use of the StringHolder class for the targetSlot, which defers the final string value resolution until runtime via the BuilderSupport context.

## Lifecycle & Ownership
-   **Creation:** Instantiated automatically by the server's asset loading system when it encounters an action of type "FlockSetTarget" within an NPC behavior JSON file. It is never created directly by game logic developers.
-   **Scope:** Extremely short-lived and transient. An instance exists only for the duration of parsing its corresponding JSON object and the subsequent call to the build method.
-   **Destruction:** The builder instance is immediately eligible for garbage collection after the build method returns the final ActionFlockSetTarget object. References to this builder must not be retained.

## Internal State & Concurrency
-   **State:** The internal state, including the clear flag and targetSlot, is mutable and is populated exclusively by the readConfig method. After this method completes, the object is treated as effectively immutable until it is destroyed.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access during the asset loading phase. Concurrent calls to readConfig or build would result in a corrupted internal state and unpredictable behavior. The asset loading pipeline must guarantee that each builder instance is confined to a single thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionFlockSetTarget | O(1) | Constructs and returns the final, immutable runtime action. |
| readConfig(JsonElement) | BuilderActionFlockSetTarget | O(N) | Configures the builder by parsing a JSON object. N is the number of keys. Returns self for chaining. |
| isClear() | boolean | O(1) | Returns the configured value of the Clear flag. |
| getTargetSlot(BuilderSupport) | String | O(1) | Resolves and returns the target slot name using the provided runtime context. |

## Integration Patterns

### Standard Usage
This class is used internally by the engine's asset loading systems. A developer will never interact with it directly. The following example illustrates the conceptual flow within the engine.

```java
// Conceptual engine code for loading an NPC action
JsonElement actionJson = parseNpcBehaviorFile(".../behavior.json");
BuilderActionFlockSetTarget builder = new BuilderActionFlockSetTarget();

// 1. Configure the builder from the asset data
builder.readConfig(actionJson);

// 2. Build the final, immutable action using a context object
BuilderSupport support = engine.getBuilderSupport();
ActionFlockSetTarget runtimeAction = builder.build(support);

// 3. Integrate the action into the NPC's behavior tree
npcBehaviorTree.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call new BuilderActionFlockSetTarget in game logic. NPC behaviors are defined entirely within JSON assets.
-   **Retaining References:** Do not store instances of this builder. It is transient configuration data. Cache the resulting ActionFlockSetTarget object produced by the build method instead.
-   **Concurrent Modification:** Do not access a builder instance from multiple threads. The asset pipeline guarantees single-threaded construction.

## Data Pipeline
The flow of data from configuration to runtime execution is strictly defined.

> Flow:
> NPC Behavior JSON Asset -> Engine JSON Deserializer -> **BuilderActionFlockSetTarget.readConfig()** -> **BuilderActionFlockSetTarget.build()** -> ActionFlockSetTarget Instance -> NPC Behavior Tree Execution


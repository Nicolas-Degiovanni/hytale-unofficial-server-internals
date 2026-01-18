---
description: Architectural reference for BuilderActionTimerModify
---

# BuilderActionTimerModify

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionTimerModify extends BuilderActionTimer {
```

## Architecture & Concepts

The BuilderActionTimerModify class is a data-driven component within the Hytale NPC asset system. It serves as a concrete implementation of the Builder pattern, designed specifically to translate a JSON configuration into a runtime command for modifying an existing NPC timer.

Its primary architectural role is to act as a deserializer and factory. It does not execute timer logic itself. Instead, it parses a static data definition (JSON) and uses it to construct an ActionTimer object. This resulting ActionTimer encapsulates the full description of the modification—such as setting a new value, changing the rate, or adding time—which is then executed by the NPC's core timer component at runtime.

A key design feature is the use of Holder objects, such as DoubleHolder and BooleanHolder. This pattern decouples the static asset definition from the dynamic game state. It allows designers to specify timer modifications using values that can be resolved at runtime via an ExecutionContext, enabling behaviors that adapt to factors like game difficulty or entity state.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderActionTimerModify is created by the server's NPC asset loading framework. This occurs when the system parses an NPC behavior file and encounters a timer action of type MODIFY. A central factory or registry is responsible for mapping this type to the BuilderActionTimerModify class and instantiating it.

-   **Scope:** The object's lifecycle is exceptionally brief and confined to the asset parsing phase. It exists only to be configured via its readConfig method and then immediately used to produce an ActionTimer via its build method.

-   **Destruction:** The builder is eligible for garbage collection as soon as the build method completes. It holds no persistent state and is not referenced by any long-lived game systems.

## Internal State & Concurrency

-   **State:** The internal state is **Mutable** during its configuration phase. The readConfig method populates its internal Holder fields from the source JSON. After this single-time population, the object should be treated as effectively immutable. Its state is a blueprint, not live game data.

-   **Thread Safety:** This class is **Not Thread-Safe**. It is designed for single-threaded access during the asset loading sequence. Any concurrent attempts to call readConfig or access its internal state will result in a race condition and undefined behavior.

    **Warning:** All NPC asset parsing and component building must be performed in a synchronized context or be confined to a dedicated, single-threaded stage of the server's startup or content loading process.

## API Surface

The public API is focused on configuration and object creation. Getters are used to resolve potentially dynamic values from their Holder containers at build time.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionTimer | O(1) | Constructs the final ActionTimer instance from the configured state. |
| readConfig(JsonElement) | BuilderActionTimer | O(N) | Populates the builder's state from a JSON object. N is the number of keys. |
| getTimerAction() | Timer.TimerAction | O(1) | Returns the static enum constant MODIFY, identifying the action type. |
| getIncreaseValue(BuilderSupport) | double | O(1) | Resolves and returns the value to be added to the target timer. |
| getRestartValueRange(BuilderSupport) | double[] | O(1) | Resolves and returns the new maximum value range for the target timer. |
| getRate(BuilderSupport) | double | O(1) | Resolves and returns the new tick rate for the target timer. |
| getSetValue(BuilderSupport) | double | O(1) | Resolves and returns the absolute value to set on the target timer. |
| isRepeating(BuilderSupport) | boolean | O(1) | Resolves and returns the new repeating status for the target timer. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use by gameplay programmers. It is an internal component of the asset pipeline. The framework uses it to deserialize a JSON definition and construct a runtime action.

```java
// Conceptual example of the asset loading framework's usage

// 1. A JSON configuration is parsed from an NPC asset file
JsonElement timerActionJson = NpcAssetParser.getTimerActionConfig(npcFile);

// 2. The framework instantiates the appropriate builder
BuilderActionTimerModify builder = new BuilderActionTimerModify();
builder.readConfig(timerActionJson);

// 3. At runtime, when an NPC is spawned, the action is built
BuilderSupport supportContext = new BuilderSupport(npc.getExecutionContext());
ActionTimer modifyAction = builder.build(supportContext);

// 4. The action is registered with the NPC's timer system
npc.getTimerComponent().addAction("someTimer", modifyAction);
```

### Anti-Patterns (Do NOT do this)

-   **Manual Instantiation:** Do not manually instantiate and configure this class in game logic code. Timer modifications should be defined declaratively in JSON asset files. Bypassing the asset pipeline breaks the data-driven design.

-   **Instance Reuse:** Do not reuse a single BuilderActionTimerModify instance to build multiple, distinct ActionTimer objects. Each builder is stateful and tied to a specific configuration. A new builder must be created for each JSON definition.

-   **Premature Access:** Do not call any `get...` or `is...` methods before the readConfig method has been successfully invoked. Doing so will yield default, uninitialized values and lead to incorrect behavior.

## Data Pipeline

The BuilderActionTimerModify is a critical transformation step in the data pipeline that converts static NPC definitions into live, executable server logic.

> Flow:
> NPC Definition (*.json file*) -> JSON Parser -> `JsonElement` -> **BuilderActionTimerModify**.readConfig() -> **BuilderActionTimerModify** (configured instance) -> **BuilderActionTimerModify**.build() -> `ActionTimer` (runtime object) -> NPC Timer Component Execution
---


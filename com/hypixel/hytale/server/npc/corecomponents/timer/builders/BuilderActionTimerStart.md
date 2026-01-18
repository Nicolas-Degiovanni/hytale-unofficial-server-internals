---
description: Architectural reference for BuilderActionTimerStart
---

# BuilderActionTimerStart

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionTimerStart extends BuilderActionTimer {
```

## Architecture & Concepts
The BuilderActionTimerStart class is a specialized factory within the server's NPC asset loading framework. It is responsible for deserializing a JSON configuration block that defines how to start or restart a timer within an NPC's behavior graph.

Its primary architectural role is to act as a bridge between the declarative data format (JSON) and the imperative runtime object model (the ActionTimer). It encapsulates the logic for parsing, validating, and ultimately constructing an ActionTimer instance configured with specific start values, rates, and repetition behavior.

This class is a concrete implementation of the Builder pattern, designed to be discovered and used by a higher-level asset processing system. The system identifies the action type from the configuration and delegates the construction process to the appropriate builder, in this case, BuilderActionTimerStart.

## Lifecycle & Ownership
-   **Creation:** An instance is created dynamically by the NPC asset loading system when it parses a JSON configuration for a timer action. It is not intended to be instantiated directly by game logic developers.
-   **Scope:** Short-lived and transient. An instance exists only for the duration of parsing a single JSON object and the subsequent call to its build method. Once the resulting ActionTimer is constructed and integrated into the NPC's behavior, the builder instance is no longer referenced and becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java garbage collector. There are no manual cleanup or disposal methods required.

## Internal State & Concurrency
-   **State:** The object is stateful but its state is transient. The internal fields, such as startValueRange and rate, are populated during the call to readConfig. This state is then used once during the call to build. The state should be considered immutable after the readConfig phase is complete.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be instantiated and used by a single thread during the asset loading sequence. The readConfig method mutates the internal state of the object. Concurrent access would lead to a corrupted or unpredictable configuration.

    **WARNING:** Do not share instances of this builder across multiple asset loading threads. Each configuration block must be processed by a new, dedicated builder instance.

## API Surface
The public API is primarily for internal framework use: consumption by the asset loader and the ActionTimer it constructs.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionTimer | O(1) | Constructs and returns a new ActionTimer instance based on the internal state populated by readConfig. |
| readConfig(JsonElement) | BuilderActionTimer | O(N) | Deserializes the provided JSON, populating and validating the builder's internal configuration. N is the number of keys. |
| getTimerAction() | Timer.TimerAction | O(1) | Returns the static enum Timer.TimerAction.START, identifying the type of this builder. |
| getStartValueRange(BuilderSupport) | double[] | O(1) | Retrieves the configured start value range. Intended for use by the constructed ActionTimer. |
| getRestartValueRange(BuilderSupport) | double[] | O(1) | Retrieves the configured restart value range. Intended for use by the constructed ActionTimer. |
| getRate(BuilderSupport) | double | O(1) | Retrieves the configured timer rate. Intended for use by the constructed ActionTimer. |
| isRepeating(BuilderSupport) | boolean | O(1) | Retrieves the configured repeating flag. Intended for use by the constructed ActionTimer. |

## Integration Patterns

### Standard Usage
This class is not used directly in gameplay code. It is invoked by the asset loading system. The conceptual flow within that system is as follows.

```java
// Hypothetical usage within an asset loading service
JsonElement config = parseNpcBehaviorFile(".../behavior.json");

// The system would identify the action type and instantiate the correct builder
BuilderActionTimerStart builder = new BuilderActionTimerStart();

// 1. Configure the builder from the data source
builder.readConfig(config.getAsJsonObject("myTimerAction"));

// 2. Build the runtime object using the loaded configuration
// BuilderSupport provides necessary runtime context
ActionTimer timerAction = builder.build(builderSupport);

// 3. Integrate the final object into the NPC's behavior tree
npcBehavior.addAction(timerAction);
```

### Anti-Patterns (Do NOT do this)
-   **State Re-use:** Do not call readConfig on an existing builder instance to process a new configuration. This will merge and overwrite state in unpredictable ways. Always create a new builder for each distinct JSON configuration block.
-   **Build Before Read:** Calling build before readConfig has been successfully invoked will result in an ActionTimer configured with default values (e.g., rate of 1.0, non-repeating), which will almost certainly cause incorrect game behavior.
-   **Direct Property Access:** While some getters are public, they are intended for the ActionTimer's constructor. External code should not call these getters; the configured behavior is encapsulated entirely within the built ActionTimer object.

## Data Pipeline
The BuilderActionTimerStart is a critical transformation step in the NPC behavior data pipeline, converting static data into an executable object.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> **BuilderActionTimerStart** -> ActionTimer Instance -> NPC Behavior State Machine


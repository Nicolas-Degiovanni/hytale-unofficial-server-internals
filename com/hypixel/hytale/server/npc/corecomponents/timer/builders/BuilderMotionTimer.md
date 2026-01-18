---
description: Architectural reference for BuilderMotionTimer
---

# BuilderMotionTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient Factory / Configuration Blueprint

## Definition
```java
// Signature
public abstract class BuilderMotionTimer<T extends Motion> extends BuilderMotionBase<T> {
```

## Architecture & Concepts

The BuilderMotionTimer class is an abstract base class within the server's NPC asset loading framework. It serves as a blueprint for creating timed *Motion* components from declarative JSON configurations. This class embodies the Decorator pattern; it does not define a motion itself but rather wraps another Motion definition, augmenting it with a time limit.

Its primary role is to act as a bridge between the static asset data (JSON files defining NPC behaviors) and the dynamic, runtime game objects (the Motion instances executed by an NPC's behavior tree). When the server loads NPC assets, a central BuilderManager identifies the appropriate concrete subclass of BuilderMotionTimer and uses it to parse the JSON, validate the configuration, and ultimately construct a specialized Motion object that will terminate after a specified duration.

This component is fundamental to creating behaviors that are not indefinite, such as "patrol for 10-15 seconds" or "stay in cover for 5 seconds".

### Lifecycle & Ownership

The lifecycle of a BuilderMotionTimer instance is ephemeral and tightly controlled by the asset loading system.

-   **Creation:** Instances of concrete subclasses are created exclusively by the BuilderManager during the parsing of an NPC asset file. A developer never instantiates these objects directly. The specific subclass used is determined by a type identifier within the JSON configuration.
-   **Scope:** The object's lifetime is confined to the asset loading and validation phase. It exists only to process its corresponding JSON block and produce a runtime Motion object.
-   **Destruction:** Once the final Motion object is built via the `build` method, the BuilderMotionTimer instance is no longer referenced by the asset system and becomes eligible for standard Java garbage collection. It holds no persistent state beyond the loading phase.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable during the initial configuration but should be considered immutable after. The `readConfig` method populates the `timerRange` and `motion` fields from a JSON source. These fields are not simple values but are holders (NumberArrayHolder, BuilderObjectReferenceHelper) that may contain expressions or references to be resolved later in the build process.

-   **Thread Safety:** **This class is not thread-safe.** The asset loading pipeline is designed to be a single-threaded process. All methods, particularly `readConfig`, directly mutate the internal state of the object. Concurrent access would lead to unpredictable behavior and data corruption. Do not share instances of this builder across threads.

## API Surface

The public API is designed for consumption by the internal BuilderManager and validation systems, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | BuilderMotionTimer | O(N) | Populates the builder's internal state from a JSON object. N is the number of keys in the object. **WARNING:** This method is not idempotent and should only be called once. |
| validate(...) | boolean | O(M) | Recursively validates this builder and its nested motion builder. M is the number of nodes in the nested configuration tree. Returns true if the configuration is valid. |
| getTimerRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured min/max timer duration as a two-element array. |
| getMotion(BuilderSupport) | T | O(M) | Constructs and returns the final, nested Motion object. This is the terminal operation for the builder. M is the complexity of building the nested motion. |

## Integration Patterns

### Standard Usage

A developer does not interact with this class using Java code. Instead, they define the behavior declaratively within an NPC's JSON asset file. The engine's asset loader handles the instantiation and processing.

The following is a conceptual JSON snippet demonstrating how a concrete implementation of BuilderMotionTimer would be used.

```json
// Example NPC Behavior Asset
{
  "type": "MotionTimer", // This key maps to a concrete BuilderMotionTimer subclass
  "Time": [ 5.0, 10.0 ],  // The timer will run for a random duration between 5 and 10 seconds
  "Motion": {
    "type": "MoveTo",    // The nested motion to execute
    "destination": "patrolPointA"
  }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance of a BuilderMotionTimer subclass using `new`. The asset system's BuilderManager is solely responsible for its creation and lifecycle. Direct instantiation bypasses critical initialization and registration steps.
-   **State Re-use:** Do not attempt to re-use a builder instance to parse multiple JSON objects. The internal state is not designed to be reset. Each configuration block in a JSON file results in a new, distinct builder instance.
-   **Post-Read Modification:** Modifying the internal state of the builder after `readConfig` has been called will lead to an inconsistent and invalid final Motion object.

## Data Pipeline

This component operates within the server's asset loading data pipeline, transforming declarative configuration into executable game logic.

> Flow:
> NPC JSON Asset File -> Server Asset Loader -> **BuilderMotionTimer.readConfig()** -> Validation Pass -> **BuilderMotionTimer.build()** -> Runtime MotionTimer Object -> NPC Behavior Tree Execution


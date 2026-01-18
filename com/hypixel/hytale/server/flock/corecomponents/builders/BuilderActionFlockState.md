---
description: Architectural reference for BuilderActionFlockState
---

# BuilderActionFlockState

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient Builder

## Definition
```java
// Signature
public class BuilderActionFlockState extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionFlockState class is a key component of the server-side NPC asset pipeline. It functions as a configuration-time factory for the runtime action, ActionFlockState. Its primary role is to deserialize a specific action definition from a JSON asset file into a concrete, executable game logic component.

This class acts as a bridge between static data and the live game engine. During server startup or asset loading, a high-level parser instantiates this builder, populates it with data from a JSON configuration via the readConfig method, and then invokes the build method. The resulting ActionFlockState object is then integrated into an NPC's behavior tree or state machine, ready to be executed during gameplay.

This pattern separates the data-definition phase (deserialization, validation) from the execution phase, ensuring that runtime action objects are always created in a valid and consistent state.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC asset loading system when it encounters an action of this type within a behavior definition file (JSON). The readConfig method is called immediately following instantiation to populate its state.
-   **Scope:** The object's lifetime is extremely short and confined to the asset parsing process. It is a transient object.
-   **Destruction:** Once the build method is called and the resulting ActionFlockState is returned, this builder instance has served its purpose. It holds no further references and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** This class holds mutable state within the protected final StringHolder field named state. This field is populated once during the call to readConfig. The use of StringHolder suggests the final state name may be resolved dynamically at runtime using an ExecutionContext.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and used within a single-threaded asset loading pipeline. Concurrent calls to readConfig or modification of its internal state would result in unpredictable behavior and data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionFlockState | O(1) | Factory method. Constructs and returns a new ActionFlockState instance using the internal configuration. |
| readConfig(JsonElement) | BuilderActionFlockState | O(1) | Deserializes the JSON data, validates the required "State" field, and populates the internal state. |
| getState(BuilderSupport) | String | O(1) | Resolves and returns the configured state name using the provided runtime context. |

## Integration Patterns

### Standard Usage
This class is intended to be used exclusively by the server's asset loading framework. The typical lifecycle is programmatic and not intended for direct use by gameplay logic developers.

```java
// Conceptual example within an asset parser
JsonElement actionJson = ... // "action": { "type": "FlockState", "State": "aggressive" }
BuilderActionFlockState builder = new BuilderActionFlockState();

// Configure the builder from the asset data
builder.readConfig(actionJson);

// Create the final runtime action object
ActionFlockState runtimeAction = builder.build(builderSupport);

// The runtimeAction is then added to an NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
-   **Calling build before readConfig:** Attempting to call build on a newly instantiated builder without first calling readConfig will produce an ActionFlockState with an uninitialized state. This will likely cause NullPointerExceptions or other assertion failures at runtime.
-   **Reusing Instances:** Builder instances are single-use. They are not designed to be reconfigured or have their build method called multiple times. A new builder must be created for each action defined in the asset files.
-   **Manual Instantiation in Gameplay Code:** This is a configuration-time object. Never instantiate or use BuilderActionFlockState within live gameplay systems, such as entity update ticks or event handlers.

## Data Pipeline
The flow of data demonstrates the transformation from a static asset definition to a live game object.

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> **BuilderActionFlockState.readConfig()** -> **BuilderActionFlockState.build()** -> ActionFlockState (Runtime Object) -> Stored in NPC Behavior Asset


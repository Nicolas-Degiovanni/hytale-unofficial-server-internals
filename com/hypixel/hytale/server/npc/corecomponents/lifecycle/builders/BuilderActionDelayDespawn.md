---
description: Architectural reference for BuilderActionDelayDespawn
---

# BuilderActionDelayDespawn

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionDelayDespawn extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionDelayDespawn class is a key component of the server-side NPC (Non-Player Character) configuration system. It implements the Builder pattern to construct an immutable ActionDelayDespawn object from a declarative JSON configuration.

Its primary role is to act as a deserialization and validation bridge. When the server loads an NPC's behavior profile from a JSON file, this builder is responsible for parsing the specific "DelayDespawn" action block. It validates the input data—for example, ensuring the delay time is a positive number—and populates its internal state. Once parsing is complete, the build method is called to produce a final, read-only ActionDelayDespawn instance, which is then integrated into the NPC's runtime behavior tree.

This pattern effectively decouples the complex and stateful process of configuration loading from the clean, immutable representation of the action used by the game logic at runtime. It inherits utility methods for JSON parsing and validation from its parent, BuilderActionBase.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level factory or configuration loader when it encounters a JSON object specifying a "DelayDespawn" action type. It is not intended for direct instantiation by game logic developers.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single JSON action block.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the build method is called and the resulting ActionDelayDespawn object is returned to the caller. Its lifecycle is tied entirely to the NPC asset loading process.

## Internal State & Concurrency
- **State:** This class is fundamentally stateful and mutable. Its fields, *time* and *shorten*, are populated sequentially as the readConfig method processes a JSON element. This internal state is temporary and serves only to accumulate the necessary parameters for constructing the final ActionDelayDespawn object.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be used exclusively within a single-threaded context during configuration loading. Its mutable fields lack any synchronization, and concurrent calls to readConfig would result in a corrupted or unpredictable state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getShortDescription() | String | O(1) | Returns a brief, human-readable summary of the action's purpose, likely for use in development tools. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Indicates the stability of this component, signaling to tools whether it is experimental, stable, or deprecated. |
| build(BuilderSupport) | ActionDelayDespawn | O(1) | Constructs and returns the final, immutable ActionDelayDespawn instance using the state configured via readConfig. This is the terminal operation. |
| readConfig(JsonElement) | BuilderActionDelayDespawn | O(N) | Parses the provided JSON data, validates required fields like "Time", and populates the builder's internal state. Returns itself to allow for potential chaining. |

## Integration Patterns

### Standard Usage
This class is not used directly in gameplay code. It is invoked by the NPC asset loading system. A developer would define the action in a JSON file, which the system then uses to drive the builder.

*NPC Behavior Definition (example.json)*
```json
{
  "type": "DelayDespawn",
  "Time": 5.5,
  "Shorten": true
}
```

*Internal System Usage (Conceptual)*
```java
// The system identifies the "DelayDespawn" type and gets the corresponding builder
BuilderActionDelayDespawn builder = npcActionFactory.getBuilder("DelayDespawn");

// The system passes the JSON data to the builder
builder.readConfig(jsonObject);

// The system builds the final action object
ActionDelayDespawn action = builder.build(builderSupport);

// The action is now ready to be added to an NPC's behavior queue
npc.getLifecycleController().addAction(action);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BuilderActionDelayDespawn()` in game logic. NPC behaviors should be defined in data files. The purpose of this class is deserialization, not programmatic construction of actions.
- **State Reuse:** Do not reuse a builder instance after calling `build()`. A builder should be treated as a single-use object for constructing one action instance. Reusing it can lead to unpredictable behavior from stale state.
- **Concurrent Access:** Never share an instance of this builder across multiple threads. The internal state is not protected and will be corrupted by concurrent writes.

## Data Pipeline
The flow of data through this component is linear, transforming declarative configuration into an executable game object.

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> **BuilderActionDelayDespawn** (`readConfig` populates state) -> `build()` method -> `ActionDelayDespawn` Instance -> NPC Behavior Controller


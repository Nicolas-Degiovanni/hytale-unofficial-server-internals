---
description: Architectural reference for BuilderActionTriggerSpawnBeacon
---

# BuilderActionTriggerSpawnBeacon

**Package:** com.hypixel.hytale.server.spawning.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionTriggerSpawnBeacon extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionTriggerSpawnBeacon class is a key component of the server-side NPC behavior system. It functions as a data-driven configuration parser, translating a JSON definition into a concrete, executable game object.

Its primary role is to act as a bridge between static asset definitions (JSON files) and the live, runtime AI system. It is responsible for deserializing a specific "action" block from an NPC's behavior configuration, validating the inputs, and ultimately constructing an ActionTriggerSpawnBeacon instance.

This pattern separates the concerns of data parsing and validation from the concerns of runtime execution. The builder handles the complex and error-prone task of interpreting designer-authored data, ensuring that the resulting Action object is created in a valid and consistent state. This makes the runtime code cleaner and more resilient to malformed configuration files.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level asset parsing system when it encounters an action of this type within an NPC's JSON configuration file. It is not managed by a dependency injection container.
- **Scope:** The lifecycle of a BuilderActionTriggerSpawnBeacon instance is extremely short. It exists only for the duration of parsing a single JSON object. After its build method is called to produce the final Action object, the builder is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** Handled by standard Java garbage collection. No explicit cleanup methods are required or provided.

## Internal State & Concurrency
- **State:** This object is highly **mutable**. Its internal fields, such as beaconId and range, are populated sequentially by the readConfig method. The state is transient and only serves as a temporary container for data being deserialized from JSON.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, configured, and used within the context of a single, serialized asset-loading thread. Concurrent calls to readConfig would result in a corrupted internal state.

## API Surface
The public API is designed for a two-phase process: configuration followed by construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderActionTriggerSpawnBeacon | O(N) | Parses and validates the provided JSON. Populates the builder's internal state. Throws exceptions on validation failure. |
| build(BuilderSupport support) | Action | O(1) | Constructs and returns a new ActionTriggerSpawnBeacon instance using the previously parsed state. |
| getBeaconId(BuilderSupport support) | int | O(log N) | Resolves the configured beacon asset name into its runtime integer ID. Called by the constructed Action at runtime. |
| getRange(BuilderSupport support) | int | O(1) | Retrieves the configured search range. Called by the constructed Action at runtime. |
| getTargetSlot(BuilderSupport support) | int | O(log N) | Resolves the configured target slot name into its runtime integer ID. Called by the constructed Action at runtime. |

## Integration Patterns

### Standard Usage
This class is intended to be used by the asset loading system. The typical flow is to instantiate, configure from JSON, and then build the final runtime object.

```java
// Hypothetical asset loader
JsonElement actionJson = getActionConfigFromJson(); // { "BeaconSpawn": "goblin_outpost_beacon", "Range": 50 }
BuilderSupport support = getBuilderSupportFromContext();

// 1. Instantiate the builder
BuilderActionTriggerSpawnBeacon builder = new BuilderActionTriggerSpawnBeacon();

// 2. Configure and validate from JSON
builder.readConfig(actionJson);

// 3. Build the final, immutable Action object
Action runtimeAction = builder.build(support);

// The builder instance is now discarded
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not attempt to reuse a builder instance to parse a second JSON object. Its internal state is not reset between calls to readConfig, which will lead to incorrect or merged configurations. Always create a new builder for each configuration block.
- **Building Without Configuration:** Calling build before readConfig has been successfully executed will result in an Action object with uninitialized data, which will almost certainly cause NullPointerExceptions or other fatal errors at runtime.
- **Direct State Modification:** Do not attempt to access and modify the internal fields (beaconId, range) directly. The readConfig method contains critical validation logic that would be bypassed.

## Data Pipeline
The builder sits at a critical junction between asset definition and runtime execution.

> Flow:
> NPC Behavior JSON File -> Asset Loading Service -> **BuilderActionTriggerSpawnBeacon.readConfig()** -> Validation & State Population -> **BuilderActionTriggerSpawnBeacon.build()** -> ActionTriggerSpawnBeacon Instance -> NPC Behavior Tree -> Game Engine Execution


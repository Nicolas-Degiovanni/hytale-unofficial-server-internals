---
description: Architectural reference for BuilderActionFlockBeacon
---

# BuilderActionFlockBeacon

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionFlockBeacon extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionFlockBeacon is a configuration and construction component within the server-side NPC AI framework. It embodies the Builder pattern to translate a data-driven AI definition, specified in a JSON format, into a concrete, executable game object: the ActionFlockBeacon.

Its primary role is to act as a deserializer and validator. During the server's asset loading phase, when an NPC's behavior tree is being constructed from configuration files, this builder is instantiated to handle the specific JSON block defining a "flock beacon" action. It parses properties such as the message content, target, and expiration, ensuring they conform to engine constraints before creating the final action object.

This class is a critical link in Hytale's data-driven AI design, allowing designers and developers to define complex NPC communication behaviors in simple JSON files without modifying core engine code.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level asset parsing system when it encounters a JSON configuration for this specific action type. It is not managed by a dependency injection container or service registry.
- **Scope:** The object's lifetime is extremely short and confined to the asset loading process. It exists only to parse a single JSON object and build one instance of ActionFlockBeacon.
- **Destruction:** Once the build method is called and the resulting ActionFlockBeacon is returned, the builder instance has served its purpose and is eligible for garbage collection. It holds no persistent references and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This object is fundamentally **mutable**. Its fields (message, sendTargetSlot, expirationTime) are populated sequentially as the readConfig method processes a JSON element. The state is transient and exists solely to configure the object being built.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for single-threaded use within the asset loading pipeline. Concurrent calls to readConfig on the same instance would result in a corrupted and unpredictable internal state.

## API Surface
The public API is designed for a sequential, single-pass configuration and build process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionFlockBeacon | O(1) | Constructs and returns the final ActionFlockBeacon instance using the state accumulated from readConfig. |
| readConfig(JsonElement) | BuilderActionFlockBeacon | O(N) | Parses the provided JSON, validates its fields, and populates the builder's internal state. N is the number of keys in the JSON object. |
| getMessage(BuilderSupport) | String | O(1) | Retrieves the configured message string. |
| getSendTargetSlot(BuilderSupport) | int | O(1) | Resolves the configured target slot name into its integer ID. |
| getExpirationTime() | double | O(1) | Retrieves the configured message expiration time in seconds. |

## Integration Patterns

### Standard Usage
This builder is intended to be used in a fluid, chain-like manner by an asset factory or loader. The typical sequence is instantiation, configuration, and final construction.

```java
// Hypothetical Asset Loader
BuilderSupport support = ...;
JsonElement actionJson = ...; // JSON for a "FlockBeacon" action

// 1. Instantiate the builder
// 2. Configure it from data
// 3. Build the final, immutable action object
BuilderActionFlockBeacon builder = new BuilderActionFlockBeacon();
builder.readConfig(actionJson);
ActionFlockBeacon action = builder.build(support);

// The 'action' object is now ready to be added to an NPC's behavior tree.
// The 'builder' object can now be discarded.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not reuse a BuilderActionFlockBeacon instance to build multiple actions. Its internal state is not reset between builds and will carry over, leading to incorrect behavior. Always create a new instance for each JSON configuration block.
- **Build Before Read:** Calling build before readConfig will produce an action with default or uninitialized values, which will almost certainly cause unintended AI behavior or runtime errors.
- **External State Mutation:** Do not modify the builder's fields directly after instantiation. The readConfig method contains critical validation logic that would be bypassed.

## Data Pipeline
This component operates during the server's boot-up or asset-reloading phase. It transforms declarative configuration data into an executable object.

> Flow:
> NPC Definition (*.json file*) -> Asset Loading Service -> Gson Parser -> JsonElement -> **BuilderActionFlockBeacon.readConfig()** -> **BuilderActionFlockBeacon.build()** -> ActionFlockBeacon Instance -> NPC Behavior Tree


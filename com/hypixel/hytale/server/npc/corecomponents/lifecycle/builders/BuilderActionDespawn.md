---
description: Architectural reference for BuilderActionDespawn
---

# BuilderActionDespawn

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionDespawn extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionDespawn class is a transient factory component within the server's data-driven NPC behavior system. It adheres to the Builder design pattern and serves a single, critical purpose: to translate a JSON configuration snippet into a concrete ActionDespawn instance.

This class acts as a deserialization bridge between the static NPC asset definitions (stored as JSON) and the live, executable action objects used by the NPC runtime. It is not the action itself, but rather the configurable blueprint used to construct the action during the server's asset loading phase. A central NPC asset manager will typically hold a registry of builder classes like this one, invoking the correct builder based on the action type specified in the configuration file.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by a higher-level asset parsing system. When the parser encounters an "ActionDespawn" type in an NPC's behavior definition, it instantiates this builder to handle the specific configuration block.
- **Scope:** The lifecycle of a BuilderActionDespawn instance is extremely short. It exists only for the duration of parsing a single action definition. Once its build method is called and the resulting ActionDespawn object is returned, the builder instance has served its purpose and is eligible for garbage collection.
- **Destruction:** Managed entirely by the Java garbage collector. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class holds minimal, mutable state. Its primary field, a boolean named force, is configured by the readConfig method. This state directly influences the properties of the ActionDespawn object it creates.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be instantiated and used by a single thread during a sequential asset loading process. Accessing a shared instance from multiple threads will lead to race conditions on its internal state. This is an intentional design choice, as the transient nature of the object makes concurrent access an invalid use case.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionDespawn | O(1) | Constructs and returns a new ActionDespawn instance based on the currently configured state. |
| readConfig(JsonElement) | BuilderActionDespawn | O(1) | Parses the provided JSON data to configure the builder's internal state. Returns itself for method chaining. |
| isForced() | boolean | O(1) | Returns the configured value of the force parameter. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked internally by the NPC asset loading pipeline. The conceptual flow is to instantiate, configure, and build in immediate succession.

```java
// Conceptual example within an asset loading system
JsonElement actionConfig = ... // JSON object for this action
BuilderActionDespawn builder = new BuilderActionDespawn();
builder.readConfig(actionConfig);
ActionDespawn action = builder.build(builderSupport);
// The 'builder' instance is now discarded
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not retain and re-use a BuilderActionDespawn instance to build multiple actions. Each action defined in a configuration file must be processed by a new, clean builder instance.
- **Concurrent Modification:** Do not call readConfig from one thread while another thread might be calling build on the same instance. The object's state is not protected by locks.
- **Build Before Configure:** Calling build before readConfig is a logical error. While it will not crash, it will produce an ActionDespawn with default parameters, ignoring the intended configuration from the asset file.

## Data Pipeline
This builder is a key transformation step in the data pipeline that converts static configuration into runtime game objects.

> Flow:
> NPC Definition (JSON File) -> Asset Parser -> **BuilderActionDespawn** -> ActionDespawn Instance -> NPC Behavior Controller

---


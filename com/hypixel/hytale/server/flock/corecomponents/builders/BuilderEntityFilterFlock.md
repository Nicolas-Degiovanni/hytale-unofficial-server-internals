---
description: Architectural reference for BuilderEntityFilterFlock
---

# BuilderEntityFilterFlock

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderEntityFilterFlock extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterFlock is a configuration-time component responsible for constructing an EntityFilterFlock instance from a data source, typically a JSON file. It operates within the server's NPC asset loading and behavior definition pipeline.

This class embodies the Builder pattern. Its primary role is to act as a translator between a declarative data format (JSON) and a concrete, executable game logic object (an IEntityFilter). By extending BuilderEntityFilterBase, it participates in a polymorphic system where various filter types can be defined and instantiated without hard-coding their construction logic into the core AI systems.

The ultimate product, an EntityFilterFlock, is used by higher-level NPC AI systems—such as behavior trees or target selectors—to make decisions based on the flocking status of other entities. This builder allows game designers to specify complex flock-related conditions, such as "target any entity that is a member of a flock of size 5-10," directly in configuration files.

## Lifecycle & Ownership
-   **Creation:** Instances are created dynamically by the server's asset loading framework. When the system parses an NPC behavior definition and encounters a filter of this type, it instantiates a BuilderEntityFilterFlock to handle the specific configuration block. It is not intended for manual instantiation.
-   **Scope:** The lifecycle of a BuilderEntityFilterFlock instance is extremely short and confined to the asset loading phase. It exists only to parse a JSON element and produce a single EntityFilterFlock object.
-   **Destruction:** Once the build method is called and the resulting IEntityFilter is returned to the asset loader, the builder instance is no longer referenced and becomes eligible for garbage collection. It holds no persistent state beyond the build process.

## Internal State & Concurrency
-   **State:** The internal state is entirely mutable. Fields such as flockMembership, size, and checkCanJoin are populated sequentially by the readConfig method. The state represents the complete configuration for the EntityFilterFlock object it is tasked with creating.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within a single-threaded asset loading context. Accessing an instance from multiple threads will lead to race conditions and unpredictable configuration states. The server's asset pipeline must guarantee that each builder instance is handled by only one thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(1) | Constructs and returns a new EntityFilterFlock instance based on the builder's current state. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses the provided JSON data to configure the builder's internal state. N is the number of keys in the JSON object. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling and debugging. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this builder, used by asset validation tools. |

## Integration Patterns

### Standard Usage
This class is not used directly by developers. It is invoked by the NPC asset loading system. The system identifies the builder type from configuration, instantiates it, configures it, and builds the final filter object.

```java
// PSEUDOCODE: System-level usage during asset loading
JsonElement filterConfig = parseNpcBehaviorFile(".../behavior.json");
String builderType = filterConfig.get("type").getAsString(); // e.g., "EntityFilterFlock"

// System registry finds and creates the correct builder
Builder<IEntityFilter> builder = builderRegistry.createBuilder(builderType);

// The builder configures itself from data and creates the final object
IEntityFilter filter = builder.readConfig(filterConfig).build(builderSupport);

// The 'filter' instance is now attached to an AI behavior
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new BuilderEntityFilterFlock()`. The asset system's registry is responsible for its creation. Manual instantiation bypasses the intended framework.
-   **State Re-use:** Do not call readConfig multiple times on the same instance or retain an instance after calling build. Each builder is single-use and should be discarded after it produces its object.
-   **Concurrent Modification:** Do not access a builder instance from multiple threads. All configuration and building must occur in a single, synchronized context.

## Data Pipeline
This component functions within a configuration data pipeline, not a real-time game data pipeline.

> Flow:
> NPC Behavior JSON File -> Server JSON Parser -> **BuilderEntityFilterFlock** (via `readConfig`) -> `build()` -> EntityFilterFlock Instance -> NPC Behavior Tree / AI System


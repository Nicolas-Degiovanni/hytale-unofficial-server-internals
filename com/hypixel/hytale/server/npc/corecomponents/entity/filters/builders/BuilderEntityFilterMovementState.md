---
description: Architectural reference for BuilderEntityFilterMovementState
---

# BuilderEntityFilterMovementState

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterMovementState extends BuilderEntityFilterBase {
```

## Architecture & Concepts
The BuilderEntityFilterMovementState is a component within the server-side NPC asset loading and configuration framework. It adheres to the Builder design pattern, serving as a specialized factory responsible for constructing an EntityFilterMovementState instance from a JSON configuration.

Its primary role is to act as a deserializer. During the server's asset loading phase, when an NPC's behavior definition is parsed, this builder is selected to handle a specific JSON object that defines a movement state check. It translates the declarative JSON data, for example `{ "State": "Walking" }`, into a concrete, executable game logic component—an IEntityFilter. This resulting filter is then integrated into the NPC's behavior tree or AI system, allowing the NPC to make decisions based on its current movement status (e.g., "is the entity currently walking?").

This class decouples the low-level representation of an entity filter from its high-level configuration, enabling game designers to define complex NPC behaviors in data without modifying engine code.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading system, likely through a registry that maps JSON type identifiers to builder classes. It is never created directly by game logic code.
- **Scope:** Extremely short-lived. An instance exists only for the duration of parsing a single JSON object. It is created, configured via readConfig, used once to build the final filter object, and is then immediately eligible for garbage collection.
- **Destruction:** The builder instance is discarded after the build method is called. The EntityFilterMovementState object it produces, however, is owned by the NPC's behavior definition and persists for the life of that NPC asset.

## Internal State & Concurrency
- **State:** The builder's state is **mutable**. The readConfig method populates the internal movementState field. This state is transient and exists only to configure the object that will be built. Once the build method is called, the builder's state is no longer relevant.
- **Thread Safety:** This class is **not thread-safe** and is not designed to be. It is intended for use within a single-threaded asset loading pipeline. Concurrent calls to readConfig would create a race condition on the movementState field. The framework guarantees that each builder instance is handled synchronously by a single thread.

## API Surface
The public API is designed for use by the asset building framework, not for general-purpose development.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | EntityFilterMovementState | O(1) | Constructs and returns the final filter object. Throws NullPointerException if readConfig was not called first. |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(1) | Parses the JSON input, validates it, and populates the internal movementState field. Returns itself for chaining. |
| getMovementState() | MovementState | O(1) | Internal getter used by the product (EntityFilterMovementState) during its construction. |

## Integration Patterns

### Standard Usage
A developer or game designer does not interact with this class directly. The engine's asset loading system uses it as part of a larger process. The conceptual flow within the framework is as follows:

```java
// Conceptual example of framework usage
// This code does not exist in one place but represents the process.

JsonElement filterConfig = parseNpcBehaviorFile(".../behavior.json");
String builderType = filterConfig.getAsJsonObject().get("type").getAsString();

// Framework finds the correct builder (e.g., from a Map<String, Class>)
BuilderEntityFilterBase builder = builderRegistry.createBuilder(builderType);

// Framework configures and builds the final object
builder.readConfig(filterConfig);
IEntityFilter filter = builder.build(builderSupport);

// The resulting filter is now ready to be used by the NPC's AI
npc.getBehaviorTree().addFilter(filter);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BuilderEntityFilterMovementState()`. The asset framework is responsible for its lifecycle. Direct creation bypasses the configuration system and will result in a non-functional object.
- **Calling build Before Configuration:** Invoking the build method before readConfig has been successfully called will cause a NullPointerException, as the internal movementState will be null. The framework's operational order prevents this.
- **Builder Reuse:** An instance of this builder is single-use. It is designed to configure and build exactly one filter object. Attempting to call readConfig multiple times or reuse the builder for another object will lead to unpredictable behavior.

## Data Pipeline
This builder acts as a transformation step in the NPC asset loading pipeline, converting declarative data into an executable object.

> Flow:
> NPC Behavior JSON File -> GSON Parser -> **BuilderEntityFilterMovementState** (via factory) -> EntityFilterMovementState Instance -> NPC Behavior Tree


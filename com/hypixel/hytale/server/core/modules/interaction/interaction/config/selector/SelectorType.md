---
description: Architectural reference for SelectorType
---

# SelectorType

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Utility / Factory Base

## Definition
```java
// Signature
public abstract class SelectorType implements NetworkSerializable<com.hypixel.hytale.protocol.Selector> {
```

## Architecture & Concepts
The SelectorType class is an abstract base class that serves as a factory blueprint for creating runtime Selector instances. It is a core component of the server's interaction and configuration system, providing a mechanism for polymorphic instantiation of different selection strategies.

In Hytale's architecture, game behaviors and interactions are often defined in external configuration files. A Selector defines a geometric area or a logical condition used to target entities or blocks for an interaction. SelectorType provides the bridge between the static configuration data and the live, in-game selection logic. Each concrete implementation, such as a BoxSelectorType or SphereSelectorType, represents a specific, configurable selection method.

The class leverages a powerful codec system, evident from the static CODEC and BASE_CODEC fields. This allows the server to deserialize various selector configurations from data sources (like JSON or network packets) into their corresponding SelectorType objects without hardcoding the types. This pattern is fundamental to the engine's data-driven design, promoting extensibility and modularity.

## Lifecycle & Ownership
- **Creation:** Instances of concrete SelectorType subclasses are not created directly. They are instantiated by the Hytale codec framework during server initialization when game content and configuration files are parsed. The `CodecMapCodec` acts as a registry, mapping a string identifier to the appropriate subclass.
- **Scope:** A SelectorType object is effectively an immutable configuration object. Once loaded, it persists for the entire server session. It is a stateless factory, so a single instance can be shared and reused globally.
- **Destruction:** These objects are cleaned up during server shutdown when the class loaders and associated static registries are garbage collected. There is no manual destruction process.

## Internal State & Concurrency
- **State:** The abstract SelectorType class is stateless. Concrete implementations are designed to be immutable, holding only the configuration parameters read at startup (e.g., the dimensions for a box selector). They do not maintain any mutable runtime state.
- **Thread Safety:** Instances are inherently thread-safe due to their immutability. The `newSelector` method produces a new, independent Selector object upon each invocation, preventing shared state issues. The static `CODEC` registry is populated during the single-threaded server bootstrap phase and is safe for concurrent reads thereafter.

**WARNING:** Modifying the static `CODEC` registry after server initialization is not supported and will lead to undefined behavior and severe concurrency issues.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newSelector() | Selector | O(1) | Abstract factory method. Creates a new, stateful Selector instance for runtime use. |
| CODEC | CodecMapCodec | N/A | Static registry for deserializing named SelectorType variants from a data source. |
| BASE_CODEC | BuilderCodec | N/A | Static base codec for the abstract class, enabling polymorphic deserialization. |

## Integration Patterns

### Standard Usage
The SelectorType is not intended for direct use by most game logic developers. It is primarily an infrastructural component. The engine's configuration loader uses the static `CODEC` to deserialize a configuration block into a specific SelectorType instance. This instance is then stored, often within a larger configuration object, and used to generate new Selector objects on demand.

```java
// Engine-level code (conceptual)

// 1. A configuration object holds the deserialized SelectorType
InteractionConfig config = loadInteractionConfig("my_interaction.json");
SelectorType selectorFactory = config.getSelectorType();

// 2. When the interaction is triggered, a new Selector is created
Selector runtimeSelector = selectorFactory.newSelector();

// 3. The runtime selector is used to find targets
List<Entity> targets = runtimeSelector.findTargets(world);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Concrete subclasses of SelectorType must not contain mutable state. They are configuration blueprints and must be thread-safe and reusable.
- **Bypassing the Codec:** Do not manually instantiate SelectorType subclasses in game logic. This defeats the data-driven design and creates hardcoded dependencies that are difficult to manage and modify. All instances should originate from the codec-based configuration loading pipeline.

## Data Pipeline
The primary data flow for SelectorType is during server initialization and configuration loading.

> Flow:
> Configuration File (e.g., JSON) -> Hytale Codec Engine -> **CodecMapCodec** -> Instantiated **SelectorType** (subclass) -> Stored in Configuration Object


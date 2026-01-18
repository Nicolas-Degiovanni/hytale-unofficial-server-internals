---
description: Architectural reference for ObjectiveTypeSetup
---

# ObjectiveTypeSetup

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.objectivesetup
**Type:** Polymorphic Base / Transient

## Definition
```java
// Signature
public abstract class ObjectiveTypeSetup {
```

## Architecture & Concepts
ObjectiveTypeSetup is an abstract base class that serves as a deserializable configuration blueprint for creating live Objective instances within the game world. It is a core component of Hytale's data-driven Adventure Mode system.

This class embodies a polymorphic factory pattern, where different types of objective setups (e.g., a single objective, a linear sequence of objectives) are defined in external configuration files. The static CODEC field, a CodecMapCodec, is the central mechanism for this system. It maps string identifiers found in configuration data, such as "Objective" or "ObjectiveLine", to their corresponding concrete Java implementation classes.

During the loading phase of an adventure, the engine uses this CODEC to parse the configuration and instantiate the appropriate ObjectiveTypeSetup subclass. The resulting object is not the objective itself, but a factory that holds the necessary configuration to create and initialize the live Objective when its setup method is invoked. This decouples the game logic from the specific structure and data of any given quest or objective.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly using the new keyword. They are materialized by the Hytale serialization engine by decoding configuration data (e.g., JSON files) via the static CODEC field. This process typically occurs when an adventure or zone is being loaded.
- **Scope:** The object's lifetime is transient and short-lived. It exists only during the objective initialization phase. Once its setup method has been called and the corresponding Objective has been created, the ObjectiveTypeSetup instance has served its purpose and is eligible for garbage collection.
- **Destruction:** Managed by the Java garbage collector. There are no explicit cleanup or teardown methods, as the object is a simple data holder.

## Internal State & Concurrency
- **State:** This abstract class is stateless. Its concrete subclasses hold immutable state that represents the configuration data deserialized from a file. This state includes details like which objective to start or the sequence of objectives to create.
- **Thread Safety:** This class and its subclasses are **not thread-safe** and must not be considered so. They are designed to be used exclusively on the main server thread during the world's logical tick cycle. The setup method's parameters, particularly the Store of EntityStore, are critical stateful engine components that forbid concurrent access.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getObjectiveIdToStart() | String | O(1) | Returns the identifier for the specific objective that should be activated by this setup. May return null. |
| setup(Set, UUID, UUID, Store) | Objective | O(N) | The core factory method. Instantiates, configures, and returns a live Objective instance based on the object's internal state. Throws exceptions on invalid configuration. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in Java code. Instead, they define the objective structure in a data file, which the engine then processes. The engine's internal logic for loading an adventure would resemble the following conceptual flow.

```java
// Engine-level code (conceptual)
// The CODEC is used to deserialize a config into an ObjectiveTypeSetup object.
ObjectiveTypeSetup setupConfig = loadAdventureConfig("my_quest.json");

// The engine then invokes setup to create the live game object.
Objective liveObjective = setupConfig.setup(players, questGiver, world, entityStore);

// The liveObjective is now managed by the game state.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create instances with `new SetupObjective()` or similar. The system is entirely dependent on the data-driven workflow through the CODEC. Direct instantiation bypasses validation and will lead to improperly configured or non-functional objectives.
- **State Modification:** Do not attempt to modify the state of a deserialized ObjectiveTypeSetup object via reflection or other means. These objects are intended to be immutable data carriers.
- **Reusing Instances:** The setup method should only be called once per instance. It is not idempotent and calling it multiple times will result in duplicate game objects and undefined behavior.

## Data Pipeline
The flow of data from configuration to live game object is a key aspect of this system's design.

> Flow:
> Adventure Mode Config File (JSON) -> Hytale Codec Engine -> **ObjectiveTypeSetup Instance** -> setup() Invocation -> Live Objective Instance -> World State


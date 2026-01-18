---
description: Architectural reference for BuilderBodyMotionTeleport
---

# BuilderBodyMotionTeleport

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionTeleport extends BuilderBodyMotionBase {
```

## Architecture & Concepts

The BuilderBodyMotionTeleport class is a factory component within the server-side NPC Behavior system. Its primary architectural role is to translate a declarative JSON configuration into a concrete, executable game logic object—specifically, a BodyMotionTeleport instruction.

This class follows the **Builder Pattern**. It is not a long-lived service but a transient object responsible for encapsulating the complex construction and validation logic for a single type of NPC movement. It acts as a critical bridge between the data-driven asset pipeline and the runtime behavior system.

During server initialization or asset hot-reloading, a central asset loading service parses NPC behavior files. When it encounters a JSON object defining a "teleport" motion, it instantiates this builder. The builder then consumes the JSON, validates its parameters against predefined rules (e.g., range checks, monotonic sequences), and verifies that the required engine features are enabled. This **load-time validation** is a key design principle, preventing invalid behavior configurations from causing runtime errors.

Upon successful configuration, the builder produces an immutable BodyMotionTeleport object, which is then integrated into an NPC's behavior tree or state machine.

## Lifecycle & Ownership

-   **Creation:** Instantiated dynamically by a higher-level factory or asset manager during the deserialization of an NPC behavior asset. The specific builder class is chosen based on a type identifier within the source JSON file. Developers must not instantiate this class directly.
-   **Scope:** Extremely short-lived. An instance of BuilderBodyMotionTeleport exists only for the duration of parsing a single JSON object and building one BodyMotionTeleport instance.
-   **Destruction:** The builder instance becomes eligible for garbage collection immediately after the `build` method is called and its result is assigned elsewhere. It holds no persistent state and is not retained by any system.

## Internal State & Concurrency

-   **State:** The internal state is **highly mutable** but only during its brief lifecycle. Fields such as offsetRadius, maxYOffset, and orientation are populated sequentially by the `readConfig` method. This state is a temporary container for parameters extracted from the JSON source.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to be used in a single-threaded context, typically within a serialized asset loading pipeline. All its state-mutating methods, such as `readConfig`, operate without locks or synchronization, as concurrent modification would lead to a corrupt and unpredictable final object.

## API Surface

The public contract is focused exclusively on the build process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderBodyMotionTeleport | O(k) | Populates the builder's internal state from a given JSON element, where k is the number of properties. Performs all validation and feature checks. Throws exceptions on invalid data. |
| build(BuilderSupport support) | BodyMotion | O(1) | Constructs and returns a new, immutable BodyMotionTeleport instance using the currently configured state. This is the terminal operation for the builder. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability flag for this component, indicating if it is considered Experimental, Stable, or Deprecated. |

## Integration Patterns

### Standard Usage

This builder is invoked by the engine's asset loading systems. A developer's interaction is purely through the JSON configuration files that this class is designed to parse.

```java
// Conceptual engine-level usage (not for game developers)

// 1. Asset loader identifies the motion type from JSON
String motionType = "BodyMotionTeleport"; 
BuilderBodyMotionBase builder = BuilderRegistry.getBuilderFor(motionType);

// 2. The specific builder is configured from the JSON data
JsonElement motionConfigData = getJsonDataForMotion();
builder.readConfig(motionConfigData);

// 3. The final, immutable instruction is built and added to the NPC
BodyMotion instruction = builder.build(builderSupport);
npcBehaviorTree.addInstruction(instruction);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BuilderBodyMotionTeleport()`. The asset system uses a registry to map JSON types to builder classes. Bypassing this registry will break the data-driven design of the behavior system.
-   **State Reuse:** Do not retain and reuse a builder instance to configure multiple BodyMotion objects. The builder's state is not designed to be reset, and reusing it will result in subsequent objects being built with a mix of old and new configuration data.
-   **Configuration after Build:** Do not attempt to call `readConfig` or other configuration methods after `build` has been invoked. The builder's lifecycle is considered complete after the build operation.

## Data Pipeline

The flow of data from configuration to a runtime object is linear and unidirectional.

> Flow:
> NPC Behavior JSON File -> Asset Deserializer -> `JsonElement` -> **BuilderBodyMotionTeleport.readConfig()** -> Populated Builder Instance -> **BuilderBodyMotionTeleport.build()** -> `BodyMotionTeleport` Instance -> NPC Runtime Behavior Component


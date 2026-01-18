---
description: Architectural reference for BuilderHeadMotionWatch
---

# BuilderHeadMotionWatch

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient / Factory

## Definition
```java
// Signature
public class BuilderHeadMotionWatch extends BuilderHeadMotionBase {
```

## Architecture & Concepts
The BuilderHeadMotionWatch class is a component-specific factory within the server's NPC asset definition pipeline. Its sole responsibility is to deserialize a JSON configuration block and construct a runtime instance of the HeadMotionWatch component.

This class acts as a translator between the static data representation of an NPC's behavior (defined in JSON) and its live, in-game object representation. It is part of a larger, abstract factory system where different builder classes are responsible for different types of NPC components.

The use of a DoubleHolder for the relativeTurnSpeed field, rather than a primitive double, is a key architectural choice. It allows the final turn speed value to be resolved dynamically at runtime via the ExecutionContext provided by BuilderSupport. This enables behaviors that can be modified by game state, such as buffs, debuffs, or environmental factors, without altering the core asset definition.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level asset parser when it encounters a head motion component of type "Watch" within an NPC's JSON definition file. The system dynamically selects this specific builder based on the configuration data.
- **Scope:** The lifecycle of a BuilderHeadMotionWatch instance is extremely short and confined to the asset loading process. It exists only to configure and build a single HeadMotionWatch object.
- **Destruction:** The instance is eligible for garbage collection immediately after the `build` method has been called and its result has been integrated into the parent NPC asset. It does not persist into the game loop.

## Internal State & Concurrency
- **State:** The class holds mutable state in its `relativeTurnSpeed` field. This state is populated exclusively through the `readConfig` method. The state is transient and only serves as a temporary container for configuration values before the final `HeadMotionWatch` object is constructed.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context during the asset deserialization phase. The intended operational sequence—instantiation, configuration, and building—is considered an atomic unit within the asset loader.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | HeadMotionWatch | O(1) | Constructs and returns the final runtime HeadMotionWatch component using the configured state. |
| readConfig(JsonElement) | BuilderHeadMotionWatch | O(1) | Deserializes the provided JSON data, populating the internal state. This method is the primary entry point for configuration. |
| getRelativeTurnSpeed(BuilderSupport) | double | O(1) | Resolves and returns the configured relative turn speed. Requires a BuilderSupport context. |

## Integration Patterns

### Standard Usage
This builder is not intended for direct use by gameplay programmers. It is invoked internally by the NPC asset loading system. The typical flow involves the system identifying the correct builder, passing it the relevant JSON data, and then calling build.

```java
// Hypothetical usage within an asset loading service
JsonElement headMotionJson = npcDefinition.get("headMotion");
BuilderHeadMotionWatch builder = new BuilderHeadMotionWatch();

// Configure the builder from the asset file
builder.readConfig(headMotionJson);

// Construct the runtime component
BuilderSupport support = createBuilderSupportForNpc(npc);
HeadMotionWatch runtimeComponent = builder.build(support);

// Attach the component to the NPC's behavior system
npc.setHeadMotion(runtimeComponent);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not reuse a single BuilderHeadMotionWatch instance to build multiple HeadMotionWatch components. Each component requires a fresh builder instance to avoid state contamination from previous configurations.
- **Building Without Configuration:** Calling `build` before `readConfig` will result in a component configured with default values (e.g., a turn speed of 1.0). This will likely lead to unintended NPC behavior.
- **Manual State Modification:** Do not attempt to modify the internal `relativeTurnSpeed` field directly. All configuration must flow through the `readConfig` method to ensure validation and proper handling.

## Data Pipeline
The class operates as a single step in a larger data transformation pipeline that converts static asset files into active game objects.

> Flow:
> NPC JSON Asset File -> Master Asset Parser -> **BuilderHeadMotionWatch**.readConfig() -> **BuilderHeadMotionWatch**.build() -> HeadMotionWatch Instance -> NPC Runtime Entity


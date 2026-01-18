---
description: Architectural reference for BuilderBodyMotionTestProbe
---

# BuilderBodyMotionTestProbe

**Package:** com.hypixel.hytale.server.npc.corecomponents.debug.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionTestProbe extends BuilderBodyMotionBase {
```

## Architecture & Concepts
The BuilderBodyMotionTestProbe is a specialized component within the server-side NPC asset system. It adheres to the **Builder design pattern** and is responsible for configuring and instantiating a BodyMotionTestProbe object. This class acts as a deserialization target, translating configuration data from a JSON format into a concrete Java object.

Its primary role is to support debugging and testing of NPC movement logic. By parsing specific JSON keys, it populates a set of parameters that are then used to construct a BodyMotionTestProbe. This probe component is then attached to an NPC to influence or analyze its movement behavior under specific, controlled conditions.

The class inherits from BuilderBodyMotionBase, indicating it is part of a larger, polymorphic framework for defining various NPC motion behaviors. The system likely uses a factory or a registry to select the correct builder based on a type identifier in the NPC's asset definition file. The explicit marking of its functionality as *Experimental* via getBuilderDescriptorState signals that this is a development-only tool not intended for production game logic.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderBodyMotionTestProbe is created by the NPC asset loading pipeline when it encounters the corresponding component type in an NPC's JSON definition. It is not intended for manual instantiation.
- **Scope:** The object's lifetime is extremely brief and confined to the NPC instantiation process. It exists only to parse a JSON block and execute its build method once.
- **Destruction:** The builder is not retained by any system after the build method returns. It becomes immediately eligible for garbage collection once the resulting BodyMotionTestProbe has been constructed and attached to the NPC.

## Internal State & Concurrency
- **State:** The internal state is entirely **Mutable**. Its fields, such as adjustX and snapAngle, are initialized with default values and are subsequently mutated by the readConfig method. This state represents the complete configuration for the BodyMotionTestProbe object it is tasked with building.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded, sequential context during asset loading. Concurrent calls to readConfig or access to its internal fields would result in a race condition and yield an unpredictably configured final object.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionTestProbe | O(1) | Constructs and returns a new BodyMotionTestProbe instance using the builder's current state. |
| readConfig(JsonElement) | BuilderBodyMotionTestProbe | O(N) | Parses the provided JSON data, mutating the builder's internal state. N is the number of keys. Returns the builder instance for chaining. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns a metadata flag, indicating the component's stability. For this class, it returns Experimental. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the server's internal NPC asset factory. The factory instantiates the builder, configures it from a JSON source, and immediately uses it to build the final component.

```java
// Conceptual example within an asset loader
JsonElement motionComponentJson = npcDefinition.get("bodyMotion");
String motionType = motionComponentJson.get("type").getAsString(); // e.g., "BodyMotionTestProbe"

// Factory creates the correct builder
BuilderBodyMotionBase builder = NpcComponentFactory.createMotionBuilder(motionType);

// Configuration and build
if (builder instanceof BuilderBodyMotionTestProbe) {
    BodyMotionTestProbe probe = ((BuilderBodyMotionTestProbe) builder)
        .readConfig(motionComponentJson)
        .build(builderSupport);
    
    npc.setBodyMotionComponent(probe);
}
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not reuse a builder instance to create multiple components. Its state is specific to a single configuration pass. A new builder must be created for each component.
- **Manual Configuration:** Avoid manually calling setter-like methods if they existed. The design intent is for configuration to be driven entirely by the readConfig method from a data source, ensuring consistency with the game's asset files.
- **Building Before Configuration:** Calling build before readConfig will produce a component with default, and likely incorrect, parameter values.

## Data Pipeline
The flow of data through this component is linear and part of the larger NPC loading process.

> Flow:
> NPC JSON Asset File -> Server Asset Loader -> **BuilderBodyMotionTestProbe.readConfig()** -> **BuilderBodyMotionTestProbe.build()** -> BodyMotionTestProbe Instance -> NPC Entity Component System


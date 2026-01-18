---
description: Architectural reference for BuilderBodyMotionFlock
---

# BuilderBodyMotionFlock

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionFlock extends BuilderBodyMotionBase {
```

## Architecture & Concepts
The BuilderBodyMotionFlock is a specialized factory component within the server's data-driven NPC asset system. Its sole responsibility is to interpret a specific JSON configuration block and construct a runtime BodyMotionFlock instance. This class acts as a deserializer and builder, translating declarative data from an asset file into a live, executable game object.

This pattern is fundamental to Hytale's entity design, allowing game designers to define complex NPC behaviors, such as flocking, in simple JSON files without modifying server source code. During server startup or asset hot-reloading, a central registry maps the JSON type identifier for "flocking" to this specific builder class. The builder is then invoked to instantiate the behavior component, which is subsequently integrated into an NPC's behavior tree or state machine.

The metadata methods, such as getBuilderDescriptorState, expose its status as **Experimental**. This signals to tooling and developers that the underlying flocking feature is incomplete and subject to change.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level factory, such as an NPC asset loader or behavior registry. This process is triggered when the system parses an NPC definition file and encounters a body motion component of type "flock".
- **Scope:** Extremely short-lived. An instance of BuilderBodyMotionFlock exists only for the duration of parsing a single JSON object and building one BodyMotionFlock object. It does not persist in memory after the build method is called.
- **Destruction:** The object becomes eligible for garbage collection immediately after the `build` method returns its result. It holds no external references and is not managed or pooled.

## Internal State & Concurrency
- **State:** The builder is stateful but its state is transient. The `readConfig` method mutates the internal fields of the builder (likely inherited from BuilderBodyMotionBase) to match the provided JSON data. This state is then consumed by the `build` method to construct the final BodyMotionFlock object.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within a single-threaded asset loading pipeline. Sharing a single instance of this builder across multiple threads will lead to race conditions and unpredictable behavior, as `readConfig` is a mutating operation. A new builder instance must be created for each distinct configuration block to be processed.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionFlock | O(1) | Constructs and returns a new BodyMotionFlock instance using the state previously configured by readConfig. |
| readConfig(JsonElement) | Builder<BodyMotion> | O(N) | Parses the input JSON data to configure the builder's internal state. N is the number of keys in the JSON object. Returns itself for chaining. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns metadata indicating the stability of this component. Currently returns Experimental. |

## Integration Patterns

### Standard Usage
A developer or system integrator does not typically invoke this class directly. It is used by the NPC asset loading system. The conceptual flow is as follows:

```java
// Pseudo-code representing the asset loading system's logic

// 1. The system identifies the builder for the JSON type "flock"
BuilderBodyMotionFlock builder = factory.getBuilderFor("flock");

// 2. The system passes the relevant JSON data to the builder
JsonElement flockConfigData = npcJson.get("bodyMotion");
builder.readConfig(flockConfigData);

// 3. The system invokes the build method to get the final object
BuilderSupport context = ...; // Provide necessary engine dependencies
BodyMotionFlock flockMotion = builder.build(context);

// 4. The resulting object is attached to the NPC
npc.setBodyMotion(flockMotion);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Never re-use a single BuilderBodyMotionFlock instance to build objects from multiple different JSON configurations. The internal state from the first `readConfig` call will bleed into subsequent builds, causing severe and hard-to-diagnose bugs.
- **Direct Instantiation:** Avoid `new BuilderBodyMotionFlock()` in general game logic. The NPC asset system is responsible for managing the creation of builders. Circumventing this can break asset hot-reloading and dependency injection.
- **Build Before Read:** Calling `build` before `readConfig` has been successfully invoked will produce a BodyMotionFlock object with default, uninitialized parameters. This will almost certainly result in incorrect or non-functional NPC behavior.

## Data Pipeline
The flow of data from a static asset file to a live game object passes directly through this builder.

> Flow:
> NPC_Definition.json -> Server Asset Loader -> JsonElement -> **BuilderBodyMotionFlock** -> BodyMotionFlock Instance -> NPC Behavior Component


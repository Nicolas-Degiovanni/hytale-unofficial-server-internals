---
description: Architectural reference for BuilderBodyMotionTimer
---

# BuilderBodyMotionTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionTimer extends BuilderMotionTimer<BodyMotion> {
```

## Architecture & Concepts
The BuilderBodyMotionTimer is a factory class that operates within the server-side NPC asset pipeline. Its sole responsibility is to construct a runtime instance of a BodyMotionTimer from deserialized configuration data. This class embodies a critical design pattern in the NPC system: **deferred dependency resolution**.

Rather than holding a direct instance of a BodyMotion asset, the builder holds a BuilderObjectReferenceHelper. This helper contains a reference, typically a string identifier, to the required BodyMotion. The actual BodyMotion object is not resolved until the `build` method is invoked.

This two-phase construction process is fundamental:
1.  **Deserialization Phase:** An asset loader parses an NPC definition file (e.g., JSON) and populates an instance of BuilderBodyMotionTimer. At this stage, dependencies like BodyMotion are stored only as unresolved references.
2.  **Build Phase:** The asset loader invokes the `build` method, passing a BuilderSupport context object. The builder uses this context to resolve its internal references into live, engine-ready objects, and then instantiates the final BodyMotionTimer.

This pattern decouples the configuration data from the runtime asset registry, allowing for modular NPC definitions and preventing loading-order dependency issues.

### Lifecycle & Ownership
-   **Creation:** Instantiated and populated by a higher-level deserialization service (e.g., Jackson) during the loading of an NPC asset from a definition file. It is never created manually by game logic code.
-   **Scope:** Extremely short-lived. Its existence is confined to the asset loading and construction sequence for a single NPC. It does not persist as part of the runtime NPC.
-   **Destruction:** Becomes eligible for garbage collection immediately after the `build` method returns its BodyMotionTimer product. The builder's state is not preserved.

## Internal State & Concurrency
-   **State:** Highly mutable. The builder acts as a temporary data container whose fields are populated by the asset deserializer. Its primary state is the unresolved reference to a BodyMotion asset.
-   **Thread Safety:** This class is **not thread-safe**. The entire NPC asset building pipeline is designed to be a single-threaded, synchronous operation. Concurrent access to a builder instance during the build phase will result in undefined behavior and potential race conditions during dependency resolution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionTimer | O(L) | Constructs a BodyMotionTimer. Resolves the internal BodyMotion reference using the provided BuilderSupport context. Returns null if the reference cannot be resolved. *L* represents the complexity of the asset lookup. |
| category() | Class<BodyMotion> | O(1) | Returns the class literal for BodyMotion. Used by the builder framework for type registration and validation. |

## Integration Patterns

### Standard Usage
The builder is used exclusively by the internal NPC asset factory. A developer will never interact with this class directly. The conceptual flow is as follows.

```java
// This process is handled automatically by the NPC asset loader.

// 1. A deserializer creates and populates the builder from a config file.
BuilderBodyMotionTimer builder = npcAssetDeserializer.createBuilder(configData);

// 2. The asset loader provides a context and invokes the build method.
BuilderSupport buildContext = assetLoader.getBuildContext();
BodyMotionTimer runtimeTimer = builder.build(buildContext);

// 3. The resulting timer is attached to the NPC's component system.
npc.getComponent(TimerComponent.class).addTimer(runtimeTimer);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderBodyMotionTimer()`. An instance created this way will lack the deserialized `motion` reference and will always fail to build a valid object.
-   **State Modification:** Do not manually modify the state of a builder after it has been created by the deserializer. Its state is considered a faithful representation of the source asset file.
-   **Builder Reuse:** A builder instance is stateful and designed for a single `build` operation. Caching or reusing a builder to create multiple timers is not supported and will lead to unexpected behavior.

## Data Pipeline
The class functions as a specific step in the data transformation pipeline that converts static configuration files into live game objects.

> Flow:
> NPC Definition File (JSON) -> Deserializer -> **BuilderBodyMotionTimer** -> `build()` with BuilderSupport -> BodyMotionTimer (Runtime Object) -> NPC Component System


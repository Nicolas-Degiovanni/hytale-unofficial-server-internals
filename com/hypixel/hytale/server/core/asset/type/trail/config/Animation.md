---
description: Architectural reference for Animation
---

# Animation

**Package:** com.hypixel.hytale.server.core.asset.type.trail.config
**Type:** Transient

## Definition
```java
// Signature
public class Animation {
```

## Architecture & Concepts
The Animation class is a data structure, not a service. It serves as a configuration object that defines the visual properties of a frame-based animation, specifically within the context of a trail asset. Its primary role is to be a deserialization target for the engine's asset loading system.

The most critical component of this class is the static final **CODEC** field. This `BuilderCodec` instance dictates how data from an asset file (e.g., JSON or a similar format) is mapped into an instance of this Java class. The architecture heavily relies on Hytale's `Codec` system for data-driven design, allowing designers and developers to define animation behavior in external files rather than hard-coding them.

The use of `appendInherited` within the codec's construction is significant. It indicates that Animation objects are part of a hierarchical asset system. A specific trail's animation can inherit default values from a parent or template asset, overriding only the necessary properties like `FrameRange` or `FrameLifeSpan`. This promotes reusability and simplifies the management of asset variants.

## Lifecycle & Ownership
- **Creation:** Instances of Animation are created exclusively by the Hytale `Codec` system during the asset loading pipeline. The static `CODEC` field is invoked by a higher-level `AssetManager` when it parses a trail asset file. Manual instantiation is an anti-pattern.
- **Scope:** The lifetime of an Animation object is strictly bound to the lifetime of its containing parent asset. It persists in memory only as long as the trail asset it belongs to is loaded.
- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded from memory by the `AssetManager`. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The Animation object is effectively immutable after its creation. While its fields are technically mutable, they are private and are only set once during the deserialization process managed by the `BuilderCodec`. It functions as a read-only data container for the rest of the engine.
- **Thread Safety:** This class is not thread-safe. It is a plain data object with no internal locking. However, because it is treated as immutable post-initialization, a fully constructed instance can be safely read from multiple threads. The asset loading process that constructs these objects is responsible for ensuring thread safety during creation.

## API Surface
The public API is limited to simple data accessors. The primary public contract is the static `CODEC` field, which is intended for use by the engine's serialization framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFrameSize() | Vector2i | O(1) | Returns the dimensions of a single animation frame. |
| getFrameRange() | Range | O(1) | Returns the start and end indices for the animation sequence. |
| getFrameLifeSpan() | int | O(1) | Returns the duration, typically in ticks, that each frame is displayed. |

## Integration Patterns

### Standard Usage
A developer will not interact with this class directly. Instead, they will define its properties within an asset file. The engine loads this configuration into an Animation object, which is then retrieved from the parent asset.

```java
// Conceptual example of retrieving the configuration
// The developer defines the animation properties in "my_trail.trail"

TrailAsset trail = assetManager.load("my_trail.trail");
Animation animationConfig = trail.getAnimation();

// Use the configuration data in the game's rendering or logic systems
int frameDuration = animationConfig.getFrameLifeSpan();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new Animation()`. This bypasses the entire data-driven asset pipeline and will result in an uninitialized, useless object. The engine relies on the `CODEC` to properly construct and populate instances from asset files.
- **Manual Deserialization:** While possible, developers should not invoke the `CODEC` directly. The `AssetManager` handles the lifecycle and caching of assets and their sub-components. Bypassing it can lead to memory leaks and redundant object creation.

## Data Pipeline
The Animation class is a destination point in the asset loading data pipeline. It translates structured text data from a file into a strongly-typed Java object for use by game systems.

> Flow:
> Trail Asset File -> AssetManager -> **Animation.CODEC** -> **Animation Object** -> Trail Rendering System


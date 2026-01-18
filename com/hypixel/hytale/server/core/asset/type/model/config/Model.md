---
description: Architectural reference for Model
---

# Model

**Package:** com.hypixel.hytale.server.core.asset.type.model.config
**Type:** Transient

## Definition
```java
// Signature
public class Model implements NetworkSerializable<com.hypixel.hytale.protocol.Model> {
```

## Architecture & Concepts
The Model class is a server-side, immutable data object that represents a specific, configured instance of a 3D model. It is not the raw model geometry itself, but rather a comprehensive description of all properties required to render and interact with a model in the game world. This includes scale, texture paths, animation sets, physics bounding boxes, particle emitters, and more.

This class acts as a bridge between the abstract asset system and the concrete world of networked entities. Its primary role is to be generated from a base ModelAsset (a template loaded from game files) and then be serialized into a network packet for transmission to the client. The client's rendering engine uses the data from this packet to construct and display the visual representation of an entity.

The design emphasizes immutability. A Model object is a snapshot of a configuration. To change an entity's model—for example, to change its scale or attachments—a new Model object must be created and assigned, replacing the old one. This ensures predictable state and thread safety.

## Lifecycle & Ownership
- **Creation:** Model instances are almost exclusively created via a suite of static factory methods, such as **createScaledModel** or **createUnitScaleModel**. These methods take a ModelAsset as a template and apply runtime parameters like scale or randomized attachments. Direct instantiation using the public constructor is strongly discouraged. These factories are typically invoked by higher-level systems responsible for entity creation or appearance updates.

- **Scope:** The lifetime of a Model object is tied to the component that holds it, usually a visual or physics component attached to a server-side entity. It is a transient object that persists only as long as it represents the entity's current appearance.

- **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or disposal methods. When an entity's model is changed or the entity itself is destroyed, the reference to the Model object is dropped, making it eligible for garbage collection.

## Internal State & Concurrency
- **State:** The Model class is fundamentally **immutable**. All of its descriptive fields are declared final and are set only once during construction. This guarantees that a Model's configuration cannot change after it has been created.

- **Thread Safety:** The class is **thread-safe** for read operations due to its immutable design. Any thread can safely access its properties without synchronization.
    - **Warning:** The internal `cachedPacket` field, a SoftReference, is mutable and used as a performance optimization for serialization. The write to this field inside the toPacket method is not synchronized. In a scenario where multiple threads call toPacket concurrently on the same instance, they might redundantly compute the packet. This is a benign race condition that only results in minor redundant work and does not corrupt the logical state of the object.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.Model | O(N) | Serializes the object into a network-ready packet. Lazily caches the result in a SoftReference. N is the sum of attachments, particles, etc. |
| getBoundingBox(MovementStates) | Box | O(1) | Returns the physics bounding box, dynamically selecting the standard or crouch box based on the provided entity state. |
| getEyeHeight(Ref, ComponentAccessor) | float | O(1) | Calculates the entity's eye height, adjusting for state-dependent offsets like crouching. Requires access to the entity's components. |
| toReference() | Model.ModelReference | O(1) | Creates a lightweight, serializable reference to this model's configuration, suitable for persistence. |
| createScaledModel(asset, scale, ...) | static Model | O(N) | The primary factory method for creating a configured Model instance from a base asset and a given scale. |

## Integration Patterns

### Standard Usage
The correct pattern involves retrieving a template ModelAsset from the asset system and then using a static factory method to create a specific, scaled instance. This instance is then associated with an entity, often via a component.

```java
// 1. Obtain the base asset from the asset manager.
ModelAsset skeletonAsset = ModelAsset.getAssetMap().getAsset("hytale:skeleton");

// 2. Create a configured Model instance using a factory method.
// This applies a specific scale and generates random attachments.
Model skeletonModel = Model.createRandomScaleModel(skeletonAsset);

// 3. Assign the model to an entity's component.
// The entity's RenderComponent (or similar) now holds a reference to this immutable model.
entity.get(RenderComponent.class).setModel(skeletonModel);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new Model(...)` directly. The constructor is highly complex with many parameters. This bypasses the crucial scaling logic and default value handling built into the static factory methods, leading to improperly configured and unpredictable model behavior.

- **State-unaware Bounding Box Access:** Do not call `getBoundingBox()` without arguments if the entity's state is available. This can lead to severe gameplay bugs, such as using the standing hitbox for a crouching entity, causing incorrect collision detection. Always use `getBoundingBox(movementStates)` when possible.

- **Post-creation Modification:** Do not attempt to modify a Model object after it has been created. If an entity's appearance needs to change, create a new Model instance with the desired properties and replace the old one on the relevant component.

## Data Pipeline
The Model class is a key transformation step in the asset-to-renderable-entity pipeline. It converts static, file-based asset data into a dynamic, in-world representation that can be sent to the client.

> Flow:
> ModelAsset (from JSON) -> **Model.createScaledModel()** -> **Model (Instance)** -> Entity Component -> **Model.toPacket()** -> Network Packet -> Client Renderer


---
description: Architectural reference for CombatTextUIComponentPositionAnimationEvent
---

# CombatTextUIComponentPositionAnimationEvent

**Package:** com.hypixel.hytale.server.core.modules.entityui.asset
**Type:** Transient Data Model

## Definition
```java
// Signature
public class CombatTextUIComponentPositionAnimationEvent extends CombatTextUIComponentAnimationEvent {
```

## Architecture & Concepts
The CombatTextUIComponentPositionAnimationEvent is a specialized data model class that represents a single, concrete animation instruction within the Entity UI framework. It is not a service or a manager; rather, it is a serializable data structure that defines how a combat text element should animate its position over time.

This class exists as part of a larger inheritance hierarchy under CombatTextUIComponentAnimationEvent. Its primary role is to be instantiated by the engine's asset loading and serialization system, guided by the static CODEC field. This codec defines how the engine should parse data from an asset file (e.g., JSON) and map it into a valid Java object instance.

Ultimately, this object serves as a server-side configuration blueprint. Its purpose is to be converted into a network-transmissible packet via the generatePacket method, which is then sent to the client to execute the actual visual animation.

### Lifecycle & Ownership
-   **Creation:** Instances are created almost exclusively by the Hytale serialization engine using the provided static BuilderCodec. This occurs when the server loads and parses entity UI asset definitions from game files. Direct manual instantiation is rare and typically reserved for internal engine logic.
-   **Scope:** The lifetime of an instance is ephemeral. It exists only during the asset parsing phase or at the moment an animation is triggered. It is a configuration object, not a long-lived entity.
-   **Destruction:** The object is managed by the Java Garbage Collector and becomes eligible for cleanup as soon as it is no longer referenced. This typically happens after the containing asset is fully processed or after its corresponding network packet has been generated and queued for transmission.

## Internal State & Concurrency
-   **State:** The class holds mutable state in the form of the private Vector2f positionOffset field. This field represents the target destination offset for the animation. The state is simple and intended to be set once upon creation by the codec.

-   **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal synchronization mechanisms. It is designed to be created, populated, and read by a single thread, such as the main server thread or a dedicated asset loading thread. Concurrent access is an anti-pattern and will lead to unpredictable behavior.

## API Surface
The public contract is minimal, focusing on serialization and packet generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(N) | **Static.** Defines the serialization and deserialization logic for this object. This is the primary entry point for the asset system. |
| generatePacket() | CombatTextEntityUIComponentAnimationEvent | O(1) | Creates and populates a network-ready packet based on the instance's state. Sets the packet type to Position. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in typical gameplay logic. It is part of the asset definition pipeline. A developer would define this event declaratively in a data file, which the engine then parses using the class's codec.

The following example illustrates how the *engine* interacts with the codec, not how a typical user would.

```java
// Engine-level code during asset loading
// Assume 'assetData' is a data structure parsed from a file.
BuilderCodec<CombatTextUIComponentPositionAnimationEvent> codec = CombatTextUIComponentPositionAnimationEvent.CODEC;

// The codec handles instantiation and field population.
CombatTextUIComponentPositionAnimationEvent event = codec.decode(context, assetData);

// The engine then uses the event to generate a network packet.
CombatTextEntityUIComponentAnimationEvent packet = event.generatePacket();
networkManager.sendPacket(player, packet);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not modify the positionOffset field after the object has been created by the asset loader. Treat instances as effectively immutable once configured. Modifying an instance that is part of a shared, cached asset can lead to visual bugs for all subsequent uses of that asset.
-   **Manual Instantiation:** Avoid using `new CombatTextUIComponentPositionAnimationEvent()`. This bypasses the validation and structured population logic defined in the codec, which is the designated creation mechanism.

## Data Pipeline
This class acts as an intermediate, structured representation of asset data before it is converted into a network packet.

> Flow:
> Game Asset File (e.g., JSON) -> Hytale Codec Engine -> **CombatTextUIComponentPositionAnimationEvent Instance** -> generatePacket() -> Network Packet -> Client-Side Animation Renderer


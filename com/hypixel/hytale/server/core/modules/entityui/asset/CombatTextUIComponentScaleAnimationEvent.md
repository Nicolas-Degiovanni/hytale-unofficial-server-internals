---
description: Architectural reference for CombatTextUIComponentScaleAnimationEvent
---

# CombatTextUIComponentScaleAnimationEvent

**Package:** com.hypixel.hytale.server.core.modules.entityui.asset
**Type:** Transient

## Definition
```java
// Signature
public class CombatTextUIComponentScaleAnimationEvent extends CombatTextUIComponentAnimationEvent {
```

## Architecture & Concepts
The CombatTextUIComponentScaleAnimationEvent is a data model class that defines a specific animation behavior for in-game combat text. It represents a "scale" transformation, where a UI element (like a damage number) changes size over a set duration.

Architecturally, this class serves as a critical link between the server's asset configuration system and the client-server network protocol. It is not a service or manager, but rather a serializable data structure. Its primary role is to be deserialized from a configuration file (e.g., JSON) via its static CODEC field, and then later transform itself into a network-ready packet using the generatePacket method.

This design pattern decouples the game's content and design (defined in asset files) from the low-level implementation of the network protocol, allowing designers to specify complex UI animations without modifying engine code.

### Lifecycle & Ownership
- **Creation:** Instances are created by the Hytale asset loading system. The static BuilderCodec, named CODEC, is invoked to parse a corresponding data block from an asset file and instantiate this object. It is not intended for direct, programmatic instantiation during gameplay.
- **Scope:** The object's lifetime is tied to its parent asset, typically a larger UI component definition. It persists in memory as part of a cached asset configuration for as long as that asset is loaded.
- **Destruction:** The object is eligible for garbage collection when the server unloads the asset bundle it belongs to, for example, during a world change or server shutdown.

## Internal State & Concurrency
- **State:** This class holds a mutable state consisting of a startScale and an endScale. These values are populated once during the deserialization process. While technically mutable, instances should be treated as immutable after initial loading to prevent inconsistent behavior.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container. All initialization and population via the CODEC should occur on a single thread, typically the main server thread or a dedicated asset loading thread. Subsequent reads, such as calls to generatePacket, are safe from any thread, provided the object's state is not modified concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | A static factory responsible for serializing and deserializing instances of this class from configuration data. |
| generatePacket() | CombatTextEntityUIComponentAnimationEvent | O(1) | Transforms this asset-level definition into a concrete network packet object for transmission to the client. |

## Integration Patterns

### Standard Usage
The primary interaction with this class is declarative, through asset files. Programmatic interaction is limited to invoking the packet generation logic on a pre-loaded asset instance.

```java
// Assume 'loadedEvent' is an instance deserialized from an asset file
CombatTextUIComponentScaleAnimationEvent loadedEvent = getEventFromAsset("my_damage_text.json");

// Generate a network packet to be sent to a client
CombatTextEntityUIComponentAnimationEvent packet = loadedEvent.generatePacket();
networkManager.sendPacket(player, packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CombatTextUIComponentScaleAnimationEvent()`. The object will be uninitialized and will fail validation or produce incorrect packets. All instances should originate from the asset pipeline via the CODEC.
- **State Mutation After Load:** Modifying the startScale or endScale fields after the asset has been loaded is a severe anti-pattern. This can lead to inconsistent UI behavior across the server and breaks the "configuration-as-code" paradigm.

## Data Pipeline
This class is a key component in the pipeline that transforms static configuration data into dynamic, client-side visual effects.

> Flow:
> Asset File (JSON/HOCON) -> Server Asset Loader -> **BuilderCodec** -> **CombatTextUIComponentScaleAnimationEvent** instance -> generatePacket() -> Network Packet -> Game Client Renderer


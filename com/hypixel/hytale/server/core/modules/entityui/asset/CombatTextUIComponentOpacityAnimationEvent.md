---
description: Architectural reference for CombatTextUIComponentOpacityAnimationEvent
---

# CombatTextUIComponentOpacityAnimationEvent

**Package:** com.hypixel.hytale.server.core.modules.entityui.asset
**Type:** Transient Data Model

## Definition
```java
// Signature
public class CombatTextUIComponentOpacityAnimationEvent extends CombatTextUIComponentAnimationEvent {
```

## Architecture & Concepts
The CombatTextUIComponentOpacityAnimationEvent is a server-side data model that defines a specific type of animation for floating combat text: a timed transition of its opacity. It represents a single, declarative instruction within a larger Entity UI asset definition.

This class is not intended for direct logical operations. Its primary role is to serve as a structured container for configuration data that is loaded from asset files (e.g., JSON). The static **CODEC** field is the central architectural feature, enabling the Hytale engine to automatically serialize and deserialize this object. This mechanism bridges the gap between human-readable asset files and the in-memory object representation used by the server.

The class inherits from CombatTextUIComponentAnimationEvent, which provides foundational properties like animation duration. This specific subclass adds the start and end opacity values required for an opacity fade.

Crucially, the **generatePacket** method transforms this asset-level definition into a network-protocol-level object. This clearly delineates its responsibility: it holds configuration on the server, and when triggered, it produces a network message to instruct the client to perform the visual animation.

## Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the Hytale **Codec** system during the asset loading phase. The engine reads a UI definition file, and the static CODEC deserializes the relevant data block into a new CombatTextUIComponentOpacityAnimationEvent object. Manual instantiation via the constructor is strongly discouraged.
-   **Scope:** This is a transient object. Its lifetime is bound to the parent UI asset that contains it. It persists in memory as long as the parent asset is loaded and referenced by the server's asset manager.
-   **Destruction:** The object is managed by the Java Garbage Collector. When the parent UI asset is unloaded and all references are cleared, this object becomes eligible for garbage collection. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The internal state is **mutable**. It consists of two float fields, startOpacity and endOpacity, which are populated during deserialization. The state is simple and contains no references to other complex systems.
-   **Thread Safety:** This class is **not thread-safe**. It is a simple data-holding object with no internal locking or synchronization. It is designed to be created and read within a single-threaded context, such as the main server thread or a dedicated asset-loading thread. Concurrent modification from multiple threads will result in data corruption and undefined behavior.

## API Surface
The public contract is minimal, focusing on its role as a data container and packet factory.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generatePacket() | CombatTextEntityUIComponentAnimationEvent | O(1) | Creates and populates a network-ready packet object from this asset's data. |
| toString() | String | O(1) | Provides a string representation for debugging and logging purposes. |
| CODEC | BuilderCodec | N/A | A static field defining the serialization and deserialization contract for this class. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly by most game logic. It is defined declaratively in an asset file and loaded by the engine. Game systems trigger the animation by referencing the parent UI component, which in turn uses this event object to generate a network packet.

```java
// CONCEPTUAL: This class is typically loaded from a file, not created in code.
// The following demonstrates how the engine might use a loaded instance.

// 1. An instance is deserialized from an asset file by the engine's Codec system.
CombatTextUIComponentOpacityAnimationEvent opacityEvent = loadedUiAsset.getAnimationEvent();

// 2. When a game event occurs, the engine generates a packet from the asset data.
CombatTextEntityUIComponentAnimationEvent packet = opacityEvent.generatePacket();

// 3. The packet is sent to the client.
networkManager.sendToClient(player, packet);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CombatTextUIComponentOpacityAnimationEvent()`. This bypasses the validation logic (e.g., range checks for opacity) and the entire asset pipeline defined by the CODEC. Always define these events in asset files.
-   **State Mutation:** Do not modify the startOpacity or endOpacity fields after the asset has been loaded. Other systems may hold a reference to this object and expect its data to be constant for the lifetime of the asset. Modifying it can lead to inconsistent visual effects for different players or subsequent triggers.

## Data Pipeline
The flow of data for this component is unidirectional, from configuration file to client-side rendering.

> Flow:
> JSON Asset File -> Server Asset Loader -> **CODEC Deserializer** -> In-Memory **CombatTextUIComponentOpacityAnimationEvent** Instance -> Game Logic Trigger -> **generatePacket()** -> Network Packet -> Client Renderer


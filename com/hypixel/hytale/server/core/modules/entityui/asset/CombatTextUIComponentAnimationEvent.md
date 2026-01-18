---
description: Architectural reference for CombatTextUIComponentAnimationEvent
---

# CombatTextUIComponentAnimationEvent

**Package:** com.hypixel.hytale.server.core.modules.entityui.asset
**Type:** Abstract Data Model / Asset

## Definition
```java
// Signature
public abstract class CombatTextUIComponentAnimationEvent
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, CombatTextUIComponentAnimationEvent>> {
```

## Architecture & Concepts
The CombatTextUIComponentAnimationEvent class is not a live, in-game component. It serves as a static, server-side data template that defines a single animation event for combat text. These templates are deserialized from JSON asset files at server startup via Hytale's powerful codec system.

Architecturally, this class acts as a bridge between configuration-as-code (JSON assets) and the network protocol. It is an abstract base class, meaning concrete implementations (e.g., FadeEvent, ScaleEvent) must be defined and registered with the codec system to handle specific animation types.

Its primary responsibility is to hold validated configuration data (e.g., start and end times) and provide a factory method, generatePacket, to translate this static data into a network-serializable protocol message. This pattern cleanly separates the asset definition from the network DTO, allowing the two to evolve independently.

## Lifecycle & Ownership
- **Creation:** Instances are never created directly with the new keyword. They are instantiated and populated exclusively by the Hytale AssetManager during the asset loading phase. The static CODEC field dictates the deserialization logic from a parent JSON map.
- **Scope:** An instance of this class persists for the entire server session. It is part of a globally accessible, read-only collection of asset definitions.
- **Destruction:** The object is garbage collected when the AssetManager unloads its asset group, which typically only occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** Effectively immutable. All properties are set once during deserialization from the asset file. There are no public setters, and the internal state is not intended to be modified at runtime. This makes the object a reliable, read-only source of configuration.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutable nature, multiple threads can safely access its properties and call its methods without any external locking or synchronization. The generatePacket method is a pure function that only reads instance state to create a new object.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique identifier for this animation event, as defined by its key in the source JSON asset. |
| generatePacket() | CombatTextEntityUIComponentAnimationEvent | O(1) | **Critical Method.** Creates and returns a new network protocol packet based on this asset's data. The returned object is of a different class in the protocol package, ready for network serialization. |

## Integration Patterns

### Standard Usage
This class is not used directly by most game logic. Instead, a higher-level system retrieves a pre-loaded instance from an asset map and uses it as a factory to generate a network packet.

```java
// Example: A combat system wants to send a "critical hit" animation
// Assume 'assetMap' is a map of all loaded CombatTextUIComponentAnimationEvent assets

// 1. Retrieve the animation definition by its ID
CombatTextUIComponentAnimationEvent animationTemplate = assetMap.get("criticalHit.scaleUp");

// 2. Generate a network-ready packet from the template
if (animationTemplate != null) {
    CombatTextEntityUIComponentAnimationEvent packet = animationTemplate.generatePacket();
    
    // 3. Send the packet to the client
    player.getNetworkConnection().send(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is impossible to instantiate this abstract class directly. Even for concrete subclasses, instantiation outside the asset pipeline is an anti-pattern, as it bypasses the asset loading and validation system.
- **State Mutation:** Do not use reflection or other means to modify the state of a loaded asset instance. These objects are shared across the entire server; runtime mutation will lead to unpredictable and non-deterministic behavior.
- **Caching Packets:** Do not cache the result of generatePacket. The method is extremely cheap, and its purpose is to create a fresh packet instance for each use case. Caching may lead to unintended state sharing if the packet object is later modified.

## Data Pipeline
The primary flow for this class is transforming static data from disk into a transient network message.

> Flow:
> JSON Asset File -> AssetManager -> **CombatTextUIComponentAnimationEvent (Deserialized Template)** -> Game Logic calls generatePacket() -> Protocol Packet -> Network Layer -> Client Render Engine


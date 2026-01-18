---
description: Architectural reference for CombatTextUIComponent
---

# CombatTextUIComponent

**Package:** com.hypixel.hytale.server.core.modules.entityui.asset
**Type:** Component/Data Model

## Definition
```java
// Signature
public class CombatTextUIComponent extends EntityUIComponent {
```

## Architecture & Concepts
The CombatTextUIComponent is a server-side data model that defines the visual properties and animation behavior of floating combat text, such as damage numbers or status effects. It is not a live, in-world object, but rather a static template or asset definition.

The core of this class's design is the static **CODEC** field. This `BuilderCodec` makes the class self-describing, enabling the server's asset loading system to deserialize configuration files (e.g., JSON) directly into a fully-populated Java object. This pattern decouples the game's asset definitions from hard-coded logic, allowing designers to configure complex UI behaviors without modifying server code.

Architecturally, this class serves as a translation layer. It converts high-level, human-readable asset definitions from disk into a low-level, optimized network packet via the `generatePacket` method. This packet is then sent to the client, which is responsible for the actual rendering and animation of the combat text.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** framework during the server's asset loading phase. The `BuilderCodec` uses the `CombatTextUIComponent::new` constructor reference to instantiate the object before populating its fields from the source data.
- **Scope:** Session-scoped. Once loaded from an asset file, an instance of CombatTextUIComponent persists in the server's asset cache for the entire server session. It is reused every time a game event requires displaying this specific type of combat text.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server's asset manager unloads its data, typically during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** The object's state is mutable *only* during the deserialization process managed by the `CODEC`. After it has been fully constructed and cached by the asset system, it must be treated as an immutable configuration object. Its fields hold all parameters required to define a combat text effect, such as duration, color, font size, and animation keyframes.
- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. It is constructed and populated on the server's main asset loading thread. Subsequent read-only operations, such as calls to `generatePacket` from the primary game loop thread, are safe.

**Warning:** Modifying the state of a cached CombatTextUIComponent instance after its initial load is an unsupported operation and will lead to unpredictable behavior for all clients.

## API Surface
The primary public contract is not its methods, but the configuration keys defined within its `CODEC`. The main programmatic interaction is to convert the object into a network packet.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generatePacket() | com.hypixel.hytale.protocol.EntityUIComponent | O(N) | Constructs and returns a network packet from the component's state. N is the number of animation events. |

## Integration Patterns

### Standard Usage
This component is not used directly. Instead, game logic requests it from an asset manager or registry by its asset key. The system then uses the component to generate a packet for network transmission.

```java
// Conceptual example of server-side combat logic
// 1. Retrieve the pre-loaded component asset
CombatTextUIComponent damageComponent = assetManager.get("hytale:combat_text_critical_hit");

// 2. Generate the network packet from the component's data
EntityUIComponent packet = damageComponent.generatePacket();

// 3. Customize packet with runtime data (e.g., the actual damage value)
// Note: This part is handled by other systems, not this class.

// 4. Send the packet to the relevant client
player.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new CombatTextUIComponent()`. Doing so creates an empty, unconfigured object that bypasses all validation and default values defined in the `CODEC`. This will result in runtime errors or visual glitches. Always retrieve instances from the asset system.
- **Post-Load Modification:** Do not attempt to get a cached instance from the asset manager and modify its fields. This is not a "live" object but a shared template. Changing it at runtime will affect every subsequent use of that combat text type across the entire server.

## Data Pipeline
The primary function of this class is to act as a specific step in the data pipeline that flows from game configuration to the player's screen.

> Flow:
> Game Asset File (JSON) -> Server Codec Deserializer -> **CombatTextUIComponent Instance** -> `generatePacket()` -> Network Packet -> Client Rendering Engine


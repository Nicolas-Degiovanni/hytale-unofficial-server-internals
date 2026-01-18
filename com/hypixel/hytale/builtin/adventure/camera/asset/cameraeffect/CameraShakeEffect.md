---
description: Architectural reference for CameraShakeEffect
---

# CameraShakeEffect

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset.cameraeffect
**Type:** Transient Data Asset

## Definition
```java
// Signature
public class CameraShakeEffect extends CameraEffect {
```

## Architecture & Concepts
The CameraShakeEffect class is a server-side data asset that defines a specific camera shake behavior. It does not implement the shake logic itself; rather, it serves as a configuration object loaded from game asset files. Its primary role is to bridge game assets with the networking layer.

This class is defined and populated by its static CODEC, a Hytale-specific serialization system. The CODEC reads asset definitions (likely JSON or HOCON) and maps fields like *CameraShake* and *Intensity* to the corresponding properties of a CameraShakeEffect instance.

During gameplay, when an event like an explosion occurs, the game logic will reference a loaded CameraShakeEffect. It then uses this object to construct a lightweight, serializable network packet containing the necessary parameters for the client to execute the visual effect. This decouples the server's game logic from the client's rendering implementation.

A key optimization is the conversion of the string-based *cameraShakeId* to an integer-based *cameraShakeIndex* in the `afterDecode` lifecycle hook. This allows the resulting network packet to be smaller and faster for the client to look up.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale asset loading system during server or client initialization. The static `CODEC` field uses the `CameraShakeEffect::new` constructor reference and then populates the instance from the asset data on disk.
-   **Scope:** An instance of CameraShakeEffect, once loaded, is effectively immutable and persists for the entire application session. It is stored and managed within a global asset registry.
-   **Destruction:** The object is garbage collected when the asset registry is cleared, typically upon server or client shutdown.

## Internal State & Concurrency
-   **State:** The object's state is mutable only during its creation by the `CODEC`. After the `afterDecode` hook completes, its state should be considered immutable. It holds configuration data, including the ID of a `CameraShake` asset, its resolved integer index, and an optional `ShakeIntensity` profile.
-   **Thread Safety:** This class is thread-safe for read operations. As a read-only data container post-initialization, multiple game threads can safely call its methods (like `createCameraShakePacket`) simultaneously without requiring locks or synchronization. All mutation is handled within the single-threaded context of the asset loading pipeline.

## API Surface
The public API is focused on interpreting the asset's configuration and creating network packets.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAccumulationMode() | AccumulationMode | O(1) | Returns how this shake should interact with other active shakes. |
| calculateIntensity(float) | float | O(1) | Applies the configured intensity modifier to a given input value. |
| createCameraShakePacket() | CameraShakeEffect | O(1) | Creates a network packet using the default intensity defined in the asset. |
| createCameraShakePacket(float) | CameraShakeEffect | O(1) | Creates a network packet using a dynamically provided intensity context, such as damage dealt or proximity to an event. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve a pre-loaded instance from an asset manager or registry and use it as a factory for network packets. The game logic provides the runtime context (e.g., intensity) to generate the final packet.

```java
// Assume 'assetManager' provides access to loaded assets
// Assume 'networkManager' sends packets to clients

// 1. Retrieve the pre-loaded effect asset by its ID
CameraShakeEffect explosionEffect = assetManager.get("hytale:strong_explosion_shake");

// 2. During a game event, calculate a context-specific intensity
float damageDealt = 50.0f;

// 3. Create the network packet using the effect as a factory
var packet = explosionEffect.createCameraShakePacket(damageDealt);

// 4. Dispatch the packet to the relevant client(s)
networkManager.sendToNearbyPlayers(packet, eventLocation);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new CameraShakeEffect()`. An instance created this way will be uninitialized, lack a `cameraShakeIndex`, and will cause NullPointerExceptions or other runtime errors when its methods are called. All instances must be managed by the asset system.
-   **State Mutation:** Do not attempt to modify the internal fields of a CameraShakeEffect after it has been loaded. The behavior of the system relies on these objects being immutable configuration singletons.

## Data Pipeline
The flow of data begins with a raw asset file and ends with a visual effect on the player's screen. This class is a critical intermediate step in that process.

> Flow:
> Asset File on Disk -> Asset Loader (with `CODEC`) -> **CameraShakeEffect** (in memory) -> Game Event Logic -> `createCameraShakePacket(float)` -> Network Packet -> Client Camera System -> Rendered Screen Shake


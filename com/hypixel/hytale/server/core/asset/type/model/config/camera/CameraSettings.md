---
description: Architectural reference for CameraSettings
---

# CameraSettings

**Package:** com.hypixel.hytale.server.core.asset.type.model.config.camera
**Type:** Transient Data Object

## Definition
```java
// Signature
public class CameraSettings implements NetworkSerializable<com.hypixel.hytale.protocol.CameraSettings> {
```

## Architecture & Concepts
CameraSettings is a data-holding class that encapsulates the configuration for a 3D camera's position and orientation relative to a target. It is not a system or a manager, but rather a plain data structure used to configure more complex camera systems.

Its primary architectural role is twofold:
1.  **Asset Deserialization Target:** The static BuilderCodec field is the most critical feature. It allows the Hytale asset pipeline to automatically deserialize camera configurations from definition files (e.g., JSON or HOCON) into a strongly-typed Java object. This class is a direct representation of camera settings defined by content creators.
2.  **Network Serialization Source:** By implementing the NetworkSerializable interface, this class acts as a bridge between the server's internal representation of an asset and the data format sent to the client. The toPacket method translates this server-side object into a protocol-level DTO, ensuring a clean separation between server logic and the network layer.

This object fundamentally represents a *state* or a *template*, not a live, ticking camera controller.

## Lifecycle & Ownership
- **Creation:** Instances are created through two primary pathways:
    1.  **Asset Pipeline:** The most common path. The static CODEC is invoked by the server's asset loading system to instantiate and populate CameraSettings objects from model or entity configuration files.
    2.  **Programmatic Instantiation:** Created directly in code via its constructors, typically for default configurations, dynamic adjustments, or cloning existing settings.
- **Scope:** The lifetime of a CameraSettings instance is bound to its owner. If it is part of a model asset, it persists as long as that asset is loaded in memory. If created dynamically for a temporary effect, its scope is transient and it will be garbage collected once its purpose is served.
- **Destruction:** Managed entirely by the Java garbage collector. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** This object is **highly mutable**. The scale method modifies the internal positionOffset vector in-place. Furthermore, the getPositionOffset method returns a direct reference to the internal Vector3f object, allowing external code to mutate the object's state without using its public API.

- **Thread Safety:** **This class is not thread-safe.** No internal locking mechanisms are present. Sharing a single instance across multiple threads is extremely dangerous and will lead to race conditions if any thread mutates its state (e.g., by calling scale or modifying the Vector3f returned by getPositionOffset).

    **WARNING:** Any instance of CameraSettings passed between threads **must** be a defensive copy, created using the clone method or the copy constructor.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.CameraSettings | O(1) | Translates this server-side object into its network protocol equivalent. |
| scale(float scale) | CameraSettings | O(1) | **Mutates** the internal positionOffset by the given scalar. Returns *this* for chaining. |
| clone() | CameraSettings | O(1) | Creates a new CameraSettings instance with a deep copy of the positionOffset. |

## Integration Patterns

### Standard Usage
CameraSettings should be treated as a configuration value. It is typically loaded from an asset and then used to configure a camera controller or an entity's view properties.

```java
// Example: Applying a cloned camera setting to a player's view
// Assume 'model' has a CameraSettings field loaded from an asset file.
CameraSettings modelCamera = model.getCameraSettings();

// CRITICAL: Clone the settings to avoid mutating the shared asset template.
CameraSettings playerCameraView = modelCamera.clone();

// Modify the instance for this specific player
playerCameraView.scale(player.getScale());

// Apply the final, unique settings to the player's camera system
player.getCameraSystem().applySettings(playerCameraView);
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Never pass a single CameraSettings instance to multiple systems that might modify it. This is the most common source of bugs with this class. Always use clone() to create a unique instance for each consumer.
    ```java
    // BAD: Both systems reference and potentially mutate the same object
    CameraSettings settings = model.getCameraSettings();
    systemA.configure(settings);
    systemB.configure(settings); // systemB may receive settings mutated by systemA
    ```
- **External Mutation:** Do not modify the Vector3f object returned by getPositionOffset. This breaks encapsulation and makes state changes difficult to track.
    ```java
    // BAD: Modifying internal state from the outside
    CameraSettings settings = new CameraSettings();
    Vector3f offset = settings.getPositionOffset();
    offset.x = 100.0f; // This directly mutates the internal state of 'settings'
    ```

## Data Pipeline
CameraSettings is a key component in the flow of configuration data from disk to the game client.

> Flow:
> Model Asset File (JSON/HOCON) -> Server Asset Loader -> **BuilderCodec deserializes to CameraSettings** -> Server Logic (e.g., Entity System) -> **toPacket() called** -> Network Protocol DTO -> Network Encoder -> Game Client


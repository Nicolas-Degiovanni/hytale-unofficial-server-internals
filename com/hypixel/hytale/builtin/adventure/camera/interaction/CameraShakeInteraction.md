---
description: Architectural reference for CameraShakeInteraction
---

# CameraShakeInteraction

**Package:** com.hypixel.hytale.builtin.adventure.camera.interaction
**Type:** Transient Data Object

## Definition
```java
// Signature
public class CameraShakeInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The CameraShakeInteraction class is a server-side, data-driven component responsible for triggering a client-side camera shake effect. It functions as a specific, concrete implementation within the server's broader Interaction System.

Architecturally, it acts as a bridge between a server-defined game event (e.g., an explosion, a creature's roar, or interacting with a specific block) and a purely visual, client-side response. It extends SimpleInstantInteraction, signifying that it represents an atomic, fire-and-forget action that completes within a single game tick and does not manage a persistent state or duration.

Its primary design feature is its reliance on the Hytale Codec system for configuration. Instances of this class are not manually created in code but are deserialized from asset files. This decouples the game logic from the specific visual effects, allowing designers to configure or change camera shake behaviors without modifying server code. When triggered, its sole responsibility is to identify the interacting player and dispatch a network packet instructing that player's client to execute the specified CameraEffect.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the `BuilderCodec` system during server startup or when game assets are loaded. The static `CODEC` field defines the deserialization logic, reading properties like the target `CameraEffect` from a configuration file.
-   **Scope:** An instance of CameraShakeInteraction is stateless and effectively immutable after creation. It persists for the entire server session, held within the asset management system as part of a larger interaction configuration.
-   **Destruction:** The object is marked for garbage collection when the server shuts down or performs a full asset reload that discards the old asset registries.

## Internal State & Concurrency
-   **State:** The internal state is minimal and becomes immutable after the `afterDecode` hook is executed by the codec.
    -   **effectId:** A String identifier for the `CameraEffect`, loaded directly from the asset file.
    -   **effectIndex:** An integer representation of the `effectId`, cached after decoding for performant lookups in the global `CameraEffect` asset map. This is a critical performance optimization to avoid repeated string-based lookups during gameplay.
-   **Thread Safety:** The object is inherently thread-safe for reads due to its immutable state post-initialization. The execution of its primary method, `firstRun`, is managed by the server's main game loop and is guaranteed to run on the appropriate thread for the world being modified. The use of a `CommandBuffer` within the `InteractionContext` ensures that any resulting entity-component system mutations are queued and processed safely at the end of the tick.

## API Surface
The primary contract is not for direct invocation by developers, but for implementation of the interaction system's lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | **Framework-Internal.** Executes the interaction logic. Retrieves the player, resolves the `CameraEffect` via the cached index, and dispatches a network packet. This method is the core of the class's runtime behavior. |
| CODEC | BuilderCodec | N/A | A static field defining how to serialize and deserialize this object from data files. It is the public contract for the asset loading system. |

## Integration Patterns

### Standard Usage
This class is not used directly in Java code. Instead, it is configured within a game asset file, such as an item or block definition. The server's interaction module will automatically instantiate and invoke it when the conditions are met.

**Example:** A hypothetical JSON asset definition.
```json
// in some_asset.json
{
  "id": "my_shaky_block",
  "components": {
    "interaction": {
      "type": "CameraShakeInteraction",
      "CameraEffect": "hytale:camera_effect_explosion_small"
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new CameraShakeInteraction()`. The object will be in an invalid state, as `effectId` and `effectIndex` will not be initialized. All instances must be created via the `CODEC` during asset loading.
-   **Manual Invocation:** Do not call the `firstRun` method directly. It relies on a fully-populated `InteractionContext` which is only provided by the server's interaction processing system during a live game tick.
-   **State Mutation:** Modifying the `effectId` or `effectIndex` fields at runtime via reflection or other means will lead to unpredictable behavior and likely cause `IndexOutOfBoundsException` or `NullPointerException` errors.

## Data Pipeline
The flow of data and control for this interaction spans from the client, to the server, and back to the client.

> Flow:
> Player Input (Client) -> Interaction Packet (Client to Server) -> Server Interaction System -> **CameraShakeInteraction.firstRun()** -> PacketHandler.writeNoCache() -> Camera Shake Packet (Server to Client) -> Client Camera System -> Visual Effect Rendered

---


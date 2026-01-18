---
description: Architectural reference for PickBlockInteraction
---

# PickBlockInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Configuration Object / Strategy

## Definition
```java
// Signature
public class PickBlockInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The PickBlockInteraction class is a specific, data-driven implementation of a client-side interaction. It represents the "pick block" or "middle-click" functionality found in many voxel-based games. Architecturally, it does not contain the core game logic itself; rather, it serves as a **network request generator**.

Its primary role within the engine's Interaction Module is to define the client's behavior when this action is triggered. When a player initiates a pick block action, the system uses an instance of this class to generate a corresponding network packet (`com.hypixel.hytale.protocol.PickBlockInteraction`). This packet is then sent to the server, which is responsible for the authoritative logic:
1.  Validating the action (e.g., checking permissions).
2.  Determining if the player is in a creative game mode.
3.  If not in creative, searching the player's inventory for the target block.
4.  If the block is found, switching the player's active hotbar slot to that item.

The static `CODEC` field is a critical component, indicating that this class is instantiated and managed by the engine's serialization system. Game designers define interactions in external configuration files, and the `BuilderCodec` is responsible for parsing those definitions and creating the corresponding Java objects.

The empty implementations of `interactWithBlock` and `simulateInteractWithBlock` are intentional. They signify that this interaction has no direct, immediate effect on the client's world state and does not support client-side prediction. The action is fully authoritative on the server.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the Hytale `Codec` system during the loading of game assets and configuration files. It is never created manually via its constructor in gameplay code.
-   **Scope:** This object is effectively a stateless singleton for its defined type. Once loaded from configuration, it persists for the entire application session, typically held in a central interaction registry.
-   **Destruction:** De-referenced and garbage collected during client shutdown when asset registries are cleared.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no mutable fields and its behavior is constant. It acts as a pure definition of a type of interaction.
-   **Thread Safety:** Inherently thread-safe due to its immutability. It can be safely read from any thread. However, the broader interaction system that uses this class is expected to operate on the main game thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns a constant `Client`, indicating the client can initiate this action without prior server approval. |
| generatePacket() | Interaction | O(1) | **Core Function.** Creates and returns a new network packet to request a pick block action from the server. |
| needsRemoteSync() | boolean | O(1) | Returns a constant `true`, signaling to the interaction system that this action must be sent to the server. |
| interactWithBlock(...) | void | O(1) | No-op. The logic is handled entirely on the server after receiving the packet. |
| simulateInteractWithBlock(...) | void | O(1) | No-op. This interaction does not support client-side prediction. |

## Integration Patterns

### Standard Usage
A developer or designer does not use this class directly in Java code. Instead, it is specified by type in a configuration file (e.g., JSON or HOCON) that defines a game interaction. The engine's interaction system handles the lifecycle.

The conceptual flow within the engine is as follows:
```java
// PSEUDOCODE: How the engine's InteractionModule might use this class.

// 1. Player input is detected (e.g., middle mouse button click).
InteractionDefinition def = interactionRegistry.get("hytale:pick_block");

// 2. The system confirms the action must be sent to the server.
if (def.needsRemoteSync()) {
    // 3. A network packet is generated using the definition object.
    Interaction packet = def.generatePacket();

    // 4. The packet is sent to the server for processing.
    networkManager.sendToServer(packet);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PickBlockInteraction()`. The class is designed to be instantiated by the configuration and codec system. Manual creation bypasses this system and will not be registered or used by the engine.
-   **Calling Logic Methods:** Calling `interactWithBlock` on a client-side instance will have no effect. The game logic is exclusively server-side. Rely on `generatePacket` to trigger the behavior.
-   **Extending for Custom Logic:** Do not extend this class to add client-side logic. If client-side effects are needed, a different base interaction class should be used, as this one is explicitly designed for a client-request/server-response pattern.

## Data Pipeline

The data flow for this component is unidirectional, from client input to the network layer.

> Flow:
> Player Input (Middle-Click) -> Interaction System -> **PickBlockInteraction** -> `generatePacket()` -> Network Packet -> Network System -> Server for Processing


---
description: Architectural reference for AnimationUtils
---

# AnimationUtils

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Utility

## Definition
```java
// Signature
public class AnimationUtils {
```

## Architecture & Concepts

AnimationUtils is a server-side, stateless utility class that acts as the primary facade for broadcasting entity animations to clients. It serves as a critical bridge between high-level game logic (such as AI behaviors, player actions, or scripted events) and the low-level networking protocol.

This class is deeply integrated into the server's Entity-Component-System (ECS) architecture. It does not operate on entity objects directly but rather on an entity's `Ref<EntityStore>` and a `ComponentAccessor`. Its core responsibility is to translate a logical request, like "play swing animation", into a `PlayAnimation` network packet. It then intelligently broadcasts this packet only to clients who can actually see the entity, using the server's visibility and culling systems via `PlayerUtil`.

The system also performs validation by checking if a requested animation exists in the entity's `ModelComponent` before broadcasting, preventing clients from receiving invalid animation requests and logging a warning on the server.

## Lifecycle & Ownership

As a static utility class, AnimationUtils has no instance lifecycle.

- **Creation:** The class is never instantiated. Its static methods are loaded into memory by the JVM ClassLoader when first referenced.
- **Scope:** Application-wide. Its methods are globally accessible throughout the server's runtime.
- **Destruction:** The class is unloaded from memory only when the server application shuts down.

## Internal State & Concurrency

- **State:** AnimationUtils is entirely stateless. It contains no member fields and all required data is provided as method arguments. Each method call is an independent, atomic operation.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, callers are responsible for ensuring that operations on the `ComponentAccessor` and the underlying `EntityStore` are performed within a thread-safe context, typically the main server game loop or a designated world thread. Concurrently modifying an entity's components while calling this utility can lead to unpredictable behavior.

## API Surface

The public API consists of overloaded static methods for playing and stopping animations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| playAnimation(ref, slot, itemAnimId, animId, sendToSelf, accessor) | void | O(P) | The canonical method to play an entity animation. Constructs and broadcasts a `PlayAnimation` packet to all P players that can see the entity. |
| stopAnimation(ref, slot, sendToSelf, accessor) | void | O(P) | Stops an animation in a specific slot. Broadcasts a `PlayAnimation` packet with a null animation ID to all P relevant players. |

**Note:** Complexity O(P) refers to the number of players P in the entity's visibility set. In a sparsely populated area, this is a fast operation. In a crowded area, the cost scales linearly with the player count.

## Integration Patterns

### Standard Usage

AnimationUtils should be invoked from server-side game logic that modifies entity state, such as within an AI behavior node, an item interaction handler, or a quest script. The caller must have access to the entity's `Ref` and a valid `ComponentAccessor`.

```java
// Example: Triggering a swing animation when a player uses an item.
// This code would exist within a server-side system.

public void onPlayerUseItem(Ref<EntityStore> playerRef, ComponentAccessor<EntityStore> accessor) {
    // Define the animation to play in the primary action slot.
    AnimationSlot slot = AnimationSlot.Action;
    String animationId = "sword_swing_1h";

    // Broadcast the animation to all nearby players, but not the player themselves
    // if client-side prediction is handling it.
    boolean sendToSelf = false;

    AnimationUtils.playAnimation(playerRef, slot, animationId, sendToSelf, accessor);
}
```

### Anti-Patterns (Do NOT do this)

- **Calling from Client Code:** This is a server-only utility. It has dependencies on server-side classes and is designed for network replication. Attempting to use it on the client will fail and represents a fundamental design error.
- **Asynchronous Modification:** Do not call `playAnimation` from a separate thread that does not have a safe handle on the ECS world state. The `ComponentAccessor` may be in an invalid state, and the entity's components (like `NetworkId`) could be read while another thread is modifying them, leading to race conditions.
- **Ignoring `sendToSelf`:** Misunderstanding the `sendToSelf` flag is a common source of bugs. If a client has predictive logic for its own animations, setting this to true can cause the animation to play twice or stutter. It should typically be false for player-controlled actions and true for server-initiated, non-predicted actions on a player entity.

## Data Pipeline

The primary function of this class is to process a high-level command into a series of low-level network packets.

> Flow:
> Server Game Logic (e.g., AI System) -> `AnimationUtils.playAnimation()` -> Component Accessor (fetches `NetworkId`, `ModelComponent`) -> `PlayAnimation` Packet Instantiation -> `PlayerUtil` (calculates visibility set) -> `PacketHandler.write()` (for each visible player) -> Network Transport Layer


---
description: Architectural reference for MovementManager
---

# MovementManager

**Package:** com.hypixel.hytale.server.core.entity.entities.player.movement
**Type:** Component (Transient)

## Definition
```java
// Signature
public class MovementManager implements Component<EntityStore> {
```

## Architecture & Concepts
The MovementManager is a server-side Entity Component System (ECS) component responsible for defining and synchronizing a player entity's physical movement capabilities. It does not execute movement logic itself; rather, it serves as a data container that configures the server-side physics simulation and communicates these parameters to the client.

This component acts as a critical bridge between high-level game configuration and the player's in-game experience. It translates abstract settings defined in game assets (MovementConfig) into a concrete, network-serializable `MovementSettings` object. This object dictates every aspect of player motion, including base speed, jump force, air control, and friction.

The core design separates a player's *base* movement profile from their *current* state. This allows game systems to apply temporary modifiers (e.g., from status effects or special zones) to the current settings without corrupting the baseline configuration loaded from assets.

## Lifecycle & Ownership
- **Creation:** A MovementManager instance is created and attached to a Player entity by the EntityStore when the player is initialized in a world. It is an integral part of the standard player entity archetype.
- **Scope:** The lifecycle of a MovementManager is strictly bound to the Player entity it is attached to. It persists as long as the player entity exists in the world.
- **Destruction:** The component is destroyed and garbage collected when its parent Player entity is removed from the EntityStore, typically when a player logs out or is unloaded from a zone.

## Internal State & Concurrency
- **State:** The component's state is mutable. It maintains two primary fields:
    - **defaultSettings:** A MovementSettings object representing the baseline configuration loaded from game assets. This should be treated as a read-only template after initialization.
    - **settings:** The active MovementSettings object used by the physics engine and sent to the client. This state is frequently modified by game logic to reflect temporary changes to player movement.

- **Thread Safety:** **WARNING:** This component is not thread-safe. As a standard ECS component, all interactions must be performed on the main server thread that owns the entity's world. Unsynchronized access from other threads will lead to race conditions, corrupted state, and client desynchronization.

## API Surface
The public API is designed for high-level, state-driven operations rather than fine-grained control.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resetDefaultsAndUpdate(ref, accessor) | void | High | Orchestrates a full refresh, reload, and network sync of movement settings. Involves asset lookup. |
| refreshDefaultSettings(ref, accessor) | void | High | Reloads the base settings from the world's current MovementConfig asset. |
| applyDefaultSettings() | void | O(k) | Resets the current settings to match the default settings, discarding any temporary modifiers. |
| update(playerPacketHandler) | void | O(k) | Serializes the current settings and sends an UpdateMovementSettings packet to the client. |
| getSettings() | MovementSettings | O(1) | Returns a reference to the currently active movement settings. |

*Complexity O(k) refers to the number of fields within the MovementSettings object.*

## Integration Patterns

### Standard Usage
The most common use case is to refresh a player's movement profile in response to a significant state change, such as changing game mode or entering a new world with different rules.

```java
// A system reacting to a player's game mode change
void onGameModeChanged(Ref<EntityStore> playerEntityRef, ComponentAccessor<EntityStore> accessor) {
    MovementManager movementManager = accessor.getComponent(playerEntityRef, MovementManager.getComponentType());

    if (movementManager != null) {
        // This single call reloads configs, resets state, and notifies the client.
        movementManager.resetDefaultsAndUpdate(playerEntityRef, accessor);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new MovementManager()`. Components must be created and managed by the EntityStore to ensure they are correctly registered within the ECS framework.
- **State Mismanagement:** Do not modify the object returned by `getDefaultSettings()`. This object is the source of truth for reset operations. All temporary modifications must be applied to the object returned by `getSettings()`.
- **Excessive Updates:** Avoid calling `update()` in a tight loop or when no settings have changed. This generates unnecessary network traffic. The `update` method should only be called after the `settings` object has been meaningfully mutated.

## Data Pipeline
The MovementManager functions as a configuration pipeline, transforming static asset data into live, networked player state.

> Flow:
> 1. **MovementConfig Asset** (Loaded from AssetStore)
> 2. -> `refreshDefaultSettings()` reads the asset and merges with other components (PhysicsValues, Player)
> 3. -> **`defaultSettings` field** (Internal baseline state)
> 4. -> `applyDefaultSettings()` copies baseline to active state
> 5. -> **`settings` field** (Mutable, active state)
> 6. -> `update()` serializes the active state
> 7. -> **UpdateMovementSettings Packet**
> 8. -> Network Layer -> Client Physics Engine


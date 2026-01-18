---
description: Architectural reference for CameraManager
---

# CameraManager

**Package:** com.hypixel.hytale.server.core.entity.entities.player
**Type:** Transient Component

## Definition
```java
// Signature
public class CameraManager implements Component<EntityStore> {
```

## Architecture & Concepts
The CameraManager is a server-side, stateful component within the Hytale Entity-Component-System (ECS) architecture. Its primary responsibility is to serve as the canonical state machine for a player's mouse and camera-related inputs. It is attached exclusively to player entities.

This component does not directly manipulate the game world or the player's camera. Instead, it acts as a data aggregator. Raw input packets from the client, such as mouse clicks and cursor movements, are processed and their state is stored within an instance of this component. Other server-side systems, such as block interaction or combat systems, then query the CameraManager on the player's entity to make gameplay decisions.

For example, when a player clicks the left mouse button to break a block, the client sends a packet. The server's network layer decodes this packet and calls *handleMouseButtonState* on the player's CameraManager instance. Later in the server tick, the BlockInteractionSystem will query this same component to see that a *Pressed* event occurred and retrieve the target block coordinates to initiate the breaking action.

The one exception to its passive nature is the *resetCamera* method, which directly dispatches a packet to the client to revert its camera to a default state. This is an explicit command, not a continuous control mechanism.

## Lifecycle & Ownership
- **Creation:** A CameraManager instance is created and attached to a player entity by the ECS framework, specifically via the EntityModule, when the player entity is first instantiated in the world. It is not intended for direct, manual instantiation.
- **Scope:** The lifecycle of a CameraManager is strictly bound to its parent player entity. It persists as long as the player entity exists within the world.
- **Destruction:** The component is marked for garbage collection and destroyed when its parent player entity is removed from the world, for instance, when a player logs out or transitions between worlds.

## Internal State & Concurrency
- **State:** The CameraManager is highly mutable. Its internal state, including mouse button states and target positions, is expected to change multiple times per second in response to player input. It functions as a volatile cache for the most recent input data.
- **Thread Safety:** **This component is not thread-safe.** All internal collections are non-synchronized. It is designed under the assumption that it will only be read from and written to by the single thread responsible for processing its parent entity's game tick. Unsynchronized access from other threads will result in race conditions, inconsistent state, and severe gameplay bugs.

**WARNING:** Any system operating on a separate thread (e.g., a physics or pathfinding worker) must not access this component directly. State should be copied from this component on the main thread before being passed to a worker thread.

## API Surface
The public API provides methods for updating and querying player input state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resetCamera(PlayerRef ref) | void | O(1) | Dispatches a network packet to the client, instructing it to reset its camera control to the default player view. |
| handleMouseButtonState(...) | void | O(1) | The primary entry point for updating state. Called by the network layer to register a mouse button press or release event. |
| getMouseButtonState(...) | MouseButtonState | O(1) | Queries the last known state of a specific mouse button. Returns Released if no state is recorded. |
| getLastMouseButtonPressedPosition(...) | Vector3i | O(1) | Retrieves the world coordinate of the block targeted when the specified mouse button was last pressed. |
| setLastScreenPoint(Vector2d) | void | O(1) | Updates the last known 2D screen coordinate of the cursor. |
| getLastTargetBlock() | Vector3i | O(1) | Retrieves the world coordinate of the last block the player's cursor was targeting. |

## Integration Patterns

### Standard Usage
A server-side gameplay system retrieves the CameraManager from a player entity to act upon the latest input. This should occur within the main server tick.

```java
// Example from a hypothetical InteractionSystem
void processPlayerInteraction(Player playerEntity) {
    CameraManager camera = playerEntity.getComponent(CameraManager.getComponentType());

    // Check if the primary mouse button was just pressed
    if (camera.getMouseButtonState(MouseButtonType.Left) == MouseButtonState.Pressed) {
        Vector3i targetBlock = camera.getLastMouseButtonPressedPosition(MouseButtonType.Left);

        if (targetBlock != null) {
            // Initiate a block interaction at the target location
            world.interactWithBlock(playerEntity, targetBlock);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new CameraManager()`. The component's lifecycle is managed entirely by the Entity-Component-System. Manually created instances will not be attached to any entity and will be ignored by game systems.
- **State Caching:** Do not read a value from the CameraManager and store it in another system for use across multiple ticks. The input state is volatile and must be queried every tick to ensure gameplay logic uses the most current data.
- **Cross-Thread Modification:** Writing to or reading from a CameraManager instance from any thread other than the entity's owner thread is strictly forbidden and will lead to data corruption.

## Data Pipeline
The CameraManager primarily acts as a sink for client-side input data and a source for server-side gameplay systems.

> **Input Flow:**
> Client Mouse Event -> Client Network Layer -> **SetPlayerInputStatePacket** -> Server Network Layer -> Packet Handler -> **CameraManager.handleMouseButtonState** (State is written)

> **Gameplay Logic Flow:**
> Server Game Tick -> Gameplay System (e.g., InteractionSystem) -> Entity.getComponent -> **CameraManager** (State is read) -> World State Mutation (e.g., break block)


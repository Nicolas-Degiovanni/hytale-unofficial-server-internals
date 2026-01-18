---
description: Architectural reference for PlayerInput
---

# PlayerInput

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Transient

## Definition
```java
// Signature
public class PlayerInput implements Component<EntityStore> {
```

## Architecture & Concepts
The PlayerInput component serves as a command buffer, decoupling raw network input from the server's entity processing systems. It is a fundamental piece of the server-side Entity Component System (ECS) for player-controlled entities.

Instead of having network packet handlers directly manipulate entity state (such as position or velocity), they instead create and enqueue immutable *InputUpdate* command objects into this component's queue. A dedicated system, operating within the main server tick, is then responsible for iterating over entities with a PlayerInput component, draining the queue, and applying the commands in a deterministic order.

This architecture provides several key benefits:
1.  **Decoupling:** The network layer is only responsible for deserializing packets and queuing commands. It does not need to know about the internal state of the physics or entity systems.
2.  **Thread Safety:** It provides a clear boundary for concurrency. The network thread(s) act as producers, adding commands to the queue. The main game thread acts as the sole consumer, processing the commands. This simplifies thread management and reduces the need for widespread locking.
3.  **Testability & Replay:** The sequence of InputUpdate commands represents a complete record of a player's actions for a given tick. This can be captured, replayed, and used for debugging, testing, or server-side replay functionality.

The component itself is pure data; all logic for command execution is encapsulated within the various implementations of the nested *InputUpdate* interface.

### Lifecycle & Ownership
-   **Creation:** An instance of PlayerInput is created and attached to an entity when that entity is designated as a player. This typically occurs when a client successfully connects and spawns into the world. The ECS framework manages this attachment.
-   **Scope:** The component's lifetime is strictly bound to its parent Player entity. It persists as long as the player remains in the world.
-   **Destruction:** The component is destroyed and garbage collected automatically by the ECS when the Player entity is removed from the world, for example, upon player disconnection.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable. The core of the component is the *inputUpdateQueue*, a list that grows and shrinks every tick as commands are added by the network layer and consumed by the entity processing system. It also maintains a simple integer *mountId* for tracking vehicle or mount state.

-   **Thread Safety:** This component is **not thread-safe** and must be handled with extreme care in a multi-threaded environment. The underlying *inputUpdateQueue* is a standard ObjectArrayList. Concurrent writes from multiple network threads or simultaneous read/write operations between the network and main game threads will lead to corruption and server instability. The engine's architecture must guarantee a single-producer, single-consumer pattern where network threads queue commands and only the main server thread dequeues and processes them.

## API Surface
The public API is minimal, focusing exclusively on queue management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| queue(InputUpdate) | void | O(1) | Enqueues a new command to be processed on a subsequent server tick. |
| getMovementUpdateQueue() | List | O(1) | Returns a reference to the internal command queue. Intended for consumption by the processing system. |
| setMountId(int) | void | O(1) | Sets the entity ID of the object this player is mounted to. |

## Integration Patterns

### Standard Usage
The primary interaction pattern involves a network packet handler creating an *InputUpdate* command and queuing it on the target player's PlayerInput component.

```java
// Example from a hypothetical network packet handler
void handlePlayerMovementPacket(Player playerEntity, PlayerMovementPacket packet) {
    // Retrieve the component from the player entity
    PlayerInput inputComponent = playerEntity.getComponent(PlayerInput.getComponentType());

    if (inputComponent != null) {
        // Create an immutable command object from packet data
        PlayerInput.SetClientVelocity command = new PlayerInput.SetClientVelocity(packet.getVelocity());

        // Queue the command for processing during the next tick
        inputComponent.queue(command);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new PlayerInput()`. Components are managed exclusively by the ECS. Attempting to manage them manually will result in components that are not tracked or processed by the game engine.
-   **External Queue Modification:** Do not add to, remove from, or clear the queue obtained from *getMovementUpdateQueue* outside of the designated entity processing system. The network layer should only use the *queue* method.
-   **Direct Command Application:** Do not call the *apply* method on an *InputUpdate* object directly. These commands must be executed by the appropriate ECS system, which provides the necessary context, such as the CommandBuffer and ArchetypeChunk, to ensure safe and correct state mutation.

## Data Pipeline
The PlayerInput component is a critical stage in the player data processing pipeline, acting as the handoff point between the asynchronous network layer and the synchronous game simulation loop.

> Flow:
> Client Input (Keystroke) -> Network Packet -> Server Network Handler -> **PlayerInput.queue(command)** -> PlayerInputSystem (Tick) -> InputUpdate.apply() -> Entity Component State Change (e.g., Velocity, TransformComponent)


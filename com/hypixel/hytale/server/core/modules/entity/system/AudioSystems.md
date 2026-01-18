---
description: Architectural reference for AudioSystems
---

# AudioSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility

## Definition
```java
// Signature
public class AudioSystems {
```

## Architecture & Concepts
The AudioSystems class is not an executable system itself but rather a static container and namespace for a collection of related Entity Component Systems (ECS). It logically groups systems responsible for processing and replicating entity-related audio on the server.

This class follows a common pattern in the engine where related functionalities are organized under a single, non-instantiable class to improve code navigation and maintain a clear modular boundary. The primary systems contained within are:

*   **EntityTrackerUpdate:** A network-focused system that replicates audio state to clients.
*   **TickMovementAudio:** A gameplay-focused system that generates procedural sounds based on entity movement.

These inner classes are the actual workers and are registered with the server's ECS scheduler.

---

# AudioSystems.EntityTrackerUpdate

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** ECS System

## Definition
```java
// Signature
public static class EntityTrackerUpdate extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
This system is a critical component of the server's networking layer, specifically for audio. It acts as the bridge between an entity's server-side audio state, represented by the AudioComponent, and the clients that need to be aware of it.

Its core responsibility is to efficiently propagate changes in an entity's audio state to all relevant players. It achieves this by integrating with the EntityTrackerSystems module, which maintains a list of players (viewers) that can currently see any given entity.

The system operates within the **EntityTrackerSystems.QUEUE_UPDATE_GROUP**. This is a crucial architectural detail, as it guarantees that this system runs *after* entity visibility has been calculated for the current tick but *before* the main network serialization stage. This ordering prevents race conditions and ensures that audio updates are only queued for clients who are confirmed to be in a state to receive them.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's primary ECS scheduler during the world initialization or server bootstrap phase. A single instance is created and registered.
-   **Scope:** The instance persists for the entire lifetime of the server world. Its tick method is invoked by the ECS scheduler once per game tick for all matching entities.
-   **Destruction:** The system instance is discarded and garbage collected when the server world is shut down and the ECS scheduler is dismantled.

## Internal State & Concurrency
-   **State:** This system is stateless. It holds immutable references to ComponentType and Query objects, which are configured at creation. All per-entity state is read from components within the ArchetypeChunk passed to the tick method.
-   **Thread Safety:** This system is designed for parallel execution. The `isParallel` method allows the ECS scheduler to distribute the workload of ticking entities across multiple worker threads. Thread safety is achieved by ensuring that the `tick` method's operations are confined to the specific entity it is processing and by writing its output (network updates) to the viewer's concurrent queue via `EntityViewer.queueUpdate`. This avoids locks and shared mutable state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(V) | Processes a single entity. If the AudioComponent is marked as dirty, it creates a ComponentUpdate packet and queues it for all V visible clients. |
| getQuery() | Query | O(1) | Returns the ECS query. Selects entities that possess both an EntityTrackerSystems.Visible and an AudioComponent. |
| getGroup() | SystemGroup | O(1) | Specifies the execution group, ensuring it runs after visibility calculation. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. It is managed entirely by the ECS scheduler. To trigger its logic, another system must modify an entity's AudioComponent and signal that it is outdated.

```java
// In another system (e.g., a combat system)
AudioComponent audio = ...;
audio.addSoundEvent(someSoundId);
// This flag is consumed by EntityTrackerUpdate to trigger a network sync
audio.setNetworkOutdated(true);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never instantiate or call the `tick` method manually. Doing so bypasses the ECS scheduler, its dependency ordering, and its parallel execution model, which will lead to unpredictable behavior and severe concurrency bugs.
-   **Incorrect Component State:** Forgetting to call `setNetworkOutdated(true)` on an AudioComponent after modifying it will result in the changes never being sent to clients.

## Data Pipeline
> Flow:
> Gameplay System modifies AudioComponent -> `consumeNetworkOutdated()` returns true -> **EntityTrackerUpdate.tick()** -> `EntityViewer.queueUpdate()` -> Network Layer serializes and sends `ComponentUpdate` packet

---

# AudioSystems.TickMovementAudio

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** ECS System

## Definition
```java
// Signature
public static class TickMovementAudio extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
TickMovementAudio is a gameplay logic system that generates procedural environmental sounds based on an entity's movement. It is responsible for effects like footstep sounds, swimming sounds, and the sound of moving through foliage.

The system works by monitoring the block type an entity is currently inside, as tracked by the PositionDataComponent. It maintains the previous block type in the MovementAudioComponent to detect state transitions. When an entity moves from one block type to another (e.g., from `air` to `water`), the system looks up the appropriate sound assets in the BlockSoundSet and triggers `MoveIn` and `MoveOut` sound events.

Unlike EntityTrackerUpdate, this system does not directly create network packets. It uses the `SoundUtil` helper, which writes sound playback commands into a `CommandBuffer`. This is a standard ECS pattern for deferring actions, allowing a later system to process all sound commands in a batch, handle prioritization, and perform the actual network replication.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS scheduler during world initialization.
-   **Scope:** Persists for the lifetime of the server world. The `tick` method is called once per game tick for every entity that matches its query.
-   **Destruction:** De-registered and garbage collected when the server world shuts down.

## Internal State & Concurrency
-   **State:** The system class itself is stateless. All state required for its logic, such as the last block type an entity was in or the cooldown for repeating sounds, is stored on a per-entity basis within the MovementAudioComponent.
-   **Thread Safety:** This system is thread-safe. It operates on a single entity at a time and writes its output to a CommandBuffer. The CommandBuffer is a thread-safe data structure designed to collect commands from multiple parallel systems, which are then executed serially later in the tick. This design avoids data races without requiring explicit locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(1) | Processes a single entity. Checks for changes in the occupied block type and handles repeating movement sounds. Writes sound commands to the CommandBuffer. |
| getQuery() | Query | O(1) | Returns the ECS query. Selects entities with TransformComponent, PositionDataComponent, MovementAudioComponent, and MovementStatesComponent. |

## Integration Patterns

### Standard Usage
This system is fully automated by the ECS. To enable movement audio for an entity, a developer must simply add the required components (especially MovementAudioComponent) to its archetype. The system will then automatically handle the logic as long as the entity's position is being updated by the physics engine.

```java
// In an entity archetype definition (e.g., JSON or code)
entity.addComponent(new TransformComponent());
entity.addComponent(new PositionDataComponent());
entity.addComponent(new MovementStatesComponent());
// Adding this component activates the TickMovementAudio system for the entity
entity.addComponent(new MovementAudioComponent());
```

### Anti-Patterns (Do NOT do this)
-   **Manual State Management:** Avoid directly modifying the internal state of a MovementAudioComponent, such as `lastInsideBlockTypeId`. This state is managed exclusively by TickMovementAudio, and external modification will break its state machine, causing sounds to play incorrectly or not at all.
-   **Missing Dependencies:** Adding a MovementAudioComponent without also adding PositionDataComponent and TransformComponent will cause the system's query to ignore the entity, and no movement sounds will be produced.

## Data Pipeline
> Flow:
> Physics System updates `PositionDataComponent.insideBlockTypeId` -> **TickMovementAudio.tick()** detects state change -> `SoundUtil.playSoundEvent3d()` -> Writes a sound command to the `CommandBuffer` -> A later system processes the CommandBuffer and replicates the sound to clients.


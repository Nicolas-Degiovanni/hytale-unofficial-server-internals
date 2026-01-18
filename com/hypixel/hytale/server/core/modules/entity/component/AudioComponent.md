---
description: Architectural reference for AudioComponent
---

# AudioComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component (Transient)

## Definition
```java
// Signature
public class AudioComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The AudioComponent is a server-side data container within Hytale's Entity-Component-System (ECS) architecture. Its primary function is to serve as a transient queue for sound events that an entity needs to emit. It does not contain logic for playing audio; rather, it holds integer identifiers that map to specific sound assets.

This component is a critical element in the server-to-client data flow for ambient and event-driven audio. A dedicated server system, likely a network synchronization or audio processing system, polls entities with this component. The component's internal dirty flag, *isNetworkOutdated*, signals to these systems that its state has changed and must be replicated to clients. This mechanism ensures that sounds are triggered on the client in response to server-side game events, such as a block breaking or a creature taking damage.

## Lifecycle & Ownership
- **Creation:** An AudioComponent is not instantiated directly. It is attached to a server-side entity by a game logic system when a sound-producing event occurs. If an entity already has an AudioComponent, the new sound is simply added to the existing instance.
- **Scope:** The component's lifetime is strictly bound to its parent entity. It persists only as long as the entity exists within the world's EntityStore.
- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed or when the component is explicitly removed from the entity. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The AudioComponent is highly mutable. Its core state consists of a list of sound event IDs and a boolean dirty flag. Both are expected to change frequently during a single game tick as various systems interact with the parent entity.
- **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified exclusively by systems operating on the main server game loop thread. Unsynchronized, multi-threaded access will result in race conditions, particularly with the *addSound* and *consumeNetworkOutdated* methods, leading to corrupted state and severe network desynchronization. All interactions must be marshaled to the main server thread.

## API Surface
The public API is minimal, focusing on state mutation and a state-consuming flag for network systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type definition for this component from the EntityModule. |
| getSoundEventIds() | int[] | O(N) | Returns a **copy** of the queued sound IDs. This is a defensive copy to prevent mutation of internal state. |
| addSound(int soundIndex) | void | Amortized O(1) | Appends a sound event ID to the internal queue and critically, sets the network dirty flag to true. |
| consumeNetworkOutdated() | boolean | O(1) | Atomically returns the current network dirty flag and resets it to false. This is the primary mechanism for synchronization systems. |
| clone() | Component | O(N) | Creates a deep copy of the component, including a new list of sound IDs. Used by the ECS framework for entity duplication or serialization. |

## Integration Patterns

### Standard Usage
The component should only be manipulated by server-side systems. The typical pattern involves retrieving or adding the component to an entity and then queuing a sound.

```java
// Example from within a server-side game logic system
Entity entity = world.getEntity(entityId);

// Get or create the component instance for this entity
AudioComponent audio = entity.getOrAddComponent(AudioComponent.class);

// Queue a sound event and mark the component for network sync
audio.addSound(SoundEvents.ENTITY_ARROW_HIT);
```

A separate network system would then poll for updated components:
```java
// Example from within a server-side network synchronization system
for (Entity entity : world.getEntitiesWith(AudioComponent.class)) {
    AudioComponent audio = entity.getComponent(AudioComponent.class);
    if (audio.consumeNetworkOutdated()) {
        // Packet construction logic...
        // Send audio.getSoundEventIds() to clients
        // Clear the server-side list after sending
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new AudioComponent()`. The ECS framework manages the component's lifecycle. Use `entity.getOrAddComponent()` to ensure proper integration with the entity store.
- **State Caching:** Do not cache a reference to an AudioComponent instance across multiple ticks. The parent entity could be destroyed, leading to a stale reference and subsequent NullPointerExceptions. Always re-fetch the component from the entity each tick.
- **Client-Side Manipulation:** This is a server-authoritative component. Attempting to create or modify it on the client will have no effect and violates the server-driven game state model.

## Data Pipeline
The AudioComponent acts as a temporary buffer in the data pipeline that transforms server-side game logic into client-side audio output.

> Flow:
> Server Game System (e.g., CombatSystem) -> **AudioComponent.addSound()** -> Network Sync System reads component state -> Server-to-Client Network Packet -> Client Network Handler -> Client-Side Audio System -> Audio Engine plays sound


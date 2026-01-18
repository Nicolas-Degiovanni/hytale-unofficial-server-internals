---
description: Architectural reference for LoadedNPCEvent
---

# LoadedNPCEvent

**Package:** com.hypixel.hytale.server.spawning
**Type:** Transient

## Definition
```java
// Signature
public class LoadedNPCEvent implements IEvent<Void> {
```

## Architecture & Concepts
The LoadedNPCEvent is a message-passing object that signals a critical state transition within the server's Non-Player Character (NPC) lifecycle. It represents the precise moment when all assets required to spawn a specific NPC type have been successfully loaded into memory and validated.

Architecturally, this event serves as a powerful decoupling mechanism. It creates a clean separation between the **Asset Loading Subsystem** and the **World Spawning Subsystem**. The asset loader can operate asynchronously, processing models, textures, and behavior trees without blocking the main game loop. Upon completion, it broadcasts this event, notifying any interested systems that a specific NPC is now "spawn-ready".

The constructor enforces a strict contract by requiring the associated builder to implement ISpawnableWithModel. This guarantees that any consumer of this event can safely assume the NPC has a physical representation and can be instantiated within the game world, preventing runtime errors related to missing models or invalid spawn data.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's asset management pipeline upon the successful and complete loading of an NPC's required assets. This is typically the final step in an asynchronous loading operation.
- **Scope:** Extremely short-lived. The event object exists only for the duration of its dispatch through the central event bus. It is a fire-and-forget signal.
- **Destruction:** The object becomes eligible for garbage collection immediately after all registered event listeners have processed it. No system should retain a long-term reference to an event instance.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal BuilderInfo reference is set once during construction and is not modified thereafter. The event acts as a read-only snapshot of the NPC's loaded state at the moment of creation.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It is safe to publish this event from a worker thread (e.g., an asset loading thread) and consume it on the main server thread without explicit locking on the event object itself.

**Warning:** While the event object is thread-safe, the systems that *handle* the event must implement their own concurrency controls. The primary use case involves a handoff from a loading thread to the main game thread, which must be managed carefully by the event bus implementation.

## API Surface
The public contract is minimal, designed for data retrieval only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBuilderInfo() | BuilderInfo | O(1) | Retrieves the data payload containing the fully-loaded NPC definition. |

## Integration Patterns

### Standard Usage
Systems responsible for populating the game world, such as a WorldSpawner or an NPCManager, should subscribe to this event. Upon receipt, they extract the BuilderInfo and use it to trigger the final spawning logic.

```java
// Example of a consumer system subscribing to the event
@Subscribe
public void onNpcAssetsReady(LoadedNPCEvent event) {
    BuilderInfo readyToSpawnInfo = event.getBuilderInfo();
    
    // The spawner can now safely use this info to create the NPC entity
    this.world.spawnEntityFromBuilder(readyToSpawnInfo);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Never create an instance of LoadedNPCEvent from general game logic. This event is strictly owned and created by the asset loading pipeline. Falsely firing this event will cause the server to attempt to spawn an NPC from incomplete or non-existent data, likely resulting in a crash.
- **State Caching:** Do not store instances of this event in a collection or field. Treat the data as ephemeral. If you need to persist the NPC definition, cache the BuilderInfo object retrieved from getBuilderInfo, not the event wrapper itself.
- **Modifying Payload:** Do not modify the state of the BuilderInfo object returned by getBuilderInfo. Consumers should treat this payload as a read-only data structure to avoid unpredictable side effects in other potential listeners.

## Data Pipeline
The LoadedNPCEvent is a key component in the data flow from asset definition to in-world entity. It acts as the signal that transitions data from a "loading" state to a "ready" state.

> Flow:
> NPC Spawn Request -> Asset Loader -> **LoadedNPCEvent** -> Server Event Bus -> NPC Spawning System -> World State Update


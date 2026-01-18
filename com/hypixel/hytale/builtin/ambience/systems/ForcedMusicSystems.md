---
description: Architectural reference for ForcedMusicSystems
---

# ForcedMusicSystems

**Package:** com.hypixel.hytale.builtin.ambience.systems
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class ForcedMusicSystems {
    // Note: This class is a non-instantiable container for two nested static System classes.
    public static class PlayerAdded extends HolderSystem<EntityStore> { /* ... */ }
    public static class Tick extends EntityTickingSystem<EntityStore> { /* ... */ }
}
```

## Architecture & Concepts

The **ForcedMusicSystems** class is a server-side container for two distinct but related systems within the Hytale Entity Component System (ECS) framework. It is not a system itself but acts as a namespace for the **PlayerAdded** and **Tick** systems. Together, these systems enforce a globally defined music track upon all connected players, overriding any ambient or biome-specific music.

This mechanism is critical for scenarios requiring a specific mood or theme, such as boss fights, cinematic events, or special zones where the game's director wishes to control the audio experience centrally.

The architecture follows a classic server-authoritative model:
1.  A global **AmbienceResource** defines the desired "forced" music track index.
2.  The **Tick** system continuously monitors this global resource.
3.  When a change is detected, it updates the per-player **AmbienceTracker** component and synchronizes the state to the client via a network packet.
4.  The **PlayerAdded** system ensures that players are correctly initialized into this system upon joining and cleaned up upon leaving.

## Lifecycle & Ownership

The systems defined within **ForcedMusicSystems** are managed entirely by the server's ECS engine. Direct lifecycle management by developers is not required and is strongly discouraged.

-   **Creation:** Instances of **ForcedMusicSystems.PlayerAdded** and **ForcedMusicSystems.Tick** are created by the ECS System Manager during server initialization and registration. The outer **ForcedMusicSystems** class is never instantiated.
-   **Scope:** These system instances are singletons that persist for the entire lifetime of the server process. They operate on entities that match their respective queries.
-   **Destruction:** The systems are destroyed and cleaned up only during a full server shutdown.

## Internal State & Concurrency

-   **State:** The outer container class is stateless. The inner systems are also fundamentally stateless, acting as processors that operate on external state. They read from the global, mutable **AmbienceResource** and read/write to the per-entity, mutable **AmbienceTracker** component. State is not stored within the system instances themselves between invocations.

-   **Thread Safety:** These systems are **not thread-safe**. The ECS engine guarantees that the **tick**, **onEntityAdd**, and **onEntityRemoved** methods are executed on the main server thread. Concurrent access from other threads will lead to race conditions and world state corruption. All interactions with components and resources must occur within the callbacks provided by the ECS framework.

## API Surface

The public contract is defined by the ECS framework's abstract system classes. These methods are callbacks invoked by the engine, not by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerAdded.onEntityAdd | void | O(1) | Engine callback. Ensures the **AmbienceTracker** component exists on new player entities. |
| PlayerAdded.onEntityRemoved | void | O(1) | Engine callback. Resets the client's music by sending a packet when a player entity is removed. |
| Tick.tick | void | O(N) | Engine callback. Iterates N matched entities, checks for music changes, and sends update packets. |
| getQuery | Query | O(1) | Engine callback. Returns the static query object used to identify relevant entities for each system. |

## Integration Patterns

### Standard Usage

These systems are designed to be automatically discovered and registered by the server's ECS engine. A developer does not interact with them directly. The primary interaction is indirect, by modifying the global **AmbienceResource**.

```java
// Correct way to trigger a global music change
// This code would exist in a different system, e.g., a boss fight manager.

// 1. Get the mutable resource from the world's store
AmbienceResource ambience = world.getStore().getResource(AmbienceResource.getResourceType());

// 2. Set the desired music index. The ForcedMusicSystems.Tick will detect this change.
ambience.setForcedMusicIndex(BOSS_FIGHT_MUSIC_INDEX);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ForcedMusicSystems.Tick()`. The ECS engine is solely responsible for creating and managing system instances. Manual creation will result in a non-functional system that is not registered to receive engine callbacks.
-   **Direct Invocation:** Never call the **tick** or **onEntityAdd** methods directly. This bypasses the engine's scheduling and state management, will likely throw exceptions, and can corrupt entity state.
-   **Stateful Implementation:** Do not add member variables to these system classes to store state between ticks. Systems should be stateless processors; all persistent data must be stored in Components or Resources.

## Data Pipeline

The primary data flow synchronizes a change in a global resource to all relevant clients.

> Flow:
> Global State Change -> **ForcedMusicSystems.Tick** -> Component Update -> Network Packet -> Client Audio Engine

1.  **Source:** An external system (e.g., quest manager, event script) modifies the **AmbienceResource** by calling **setForcedMusicIndex**.
2.  **Detection:** On a subsequent server tick, the **ForcedMusicSystems.Tick** system iterates through all entities with a **PlayerRef** and **AmbienceTracker**. It compares the global index from the resource with the index stored in the player's **AmbienceTracker**.
3.  **State Update:** If the indices differ, the system updates the player's **AmbienceTracker** component with the new index.
4.  **Synchronization:** The system immediately obtains a pooled **UpdateEnvironmentMusic** packet, sets the new music index, and writes it to the player's network connection via the **PlayerRef** component.
5.  **Client Effect:** The client receives the packet and instructs its audio engine to play the new forced music track.


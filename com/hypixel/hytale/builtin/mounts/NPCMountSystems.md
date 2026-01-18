---
description: Architectural reference for NPCMountSystems
---

# NPCMountSystems

**Package:** com.hypixel.hytale.builtin.mounts
**Type:** Utility

## Definition
```java
// Signature
public class NPCMountSystems {
   // Contains static inner classes:
   // DismountOnMountDeath
   // DismountOnPlayerDeath
   // OnAdd
}
```

## Architecture & Concepts
NPCMountSystems is not a concrete object but a static container, or namespace, for a group of highly specialized systems within the Entity Component System (ECS) framework. These inner classes collectively form the core logic for managing the lifecycle of an NPC mount relationship on the server.

This system group operates on a purely reactive, event-driven model. It does not actively poll for state changes. Instead, it subscribes to specific events within the game world—primarily the addition of certain components to entities—and executes logic in response. Its primary responsibilities are:
- **Mount Initiation:** When an NPCMountComponent is added to an entity, the OnAdd system establishes the link between the player and the NPC, updates the player's state, and sends the necessary network packet to the client.
- **Dismount on Death:** The DismountOnMountDeath and DismountOnPlayerDeath systems act as cleanup handlers. They ensure that if either the player or the mount entity dies, the mount relationship is cleanly severed, and player state is restored.
- **State Restoration:** A key function is restoring the NPC's original behavior (its "role") if the mounting process is aborted or the player dismounts.

These systems are a critical bridge between the high-level gameplay action of "mounting an NPC" and the low-level ECS state changes and network synchronization required to make it function.

### Lifecycle & Ownership
- **Creation:** The inner system classes (OnAdd, DismountOnPlayerDeath, etc.) are not instantiated directly by gameplay code. They are instantiated by the server's central System Manager during the engine's bootstrap or world-loading phase. They are discovered and registered as part of the server's core behavior.
- **Scope:** Instances of these systems persist for the entire lifetime of the server world they are registered in. They are effectively singletons within the context of a world's ECS registry.
- **Destruction:** The systems are destroyed and cleaned up only when the server world itself is unloaded.

## Internal State & Concurrency
- **State:** The NPCMountSystems container is stateless. The inner system classes are also fundamentally stateless. They do not hold or cache any data between invocations. All state they operate on is read directly from components (like NPCMountComponent or Player) in the ECS Store at the moment of execution.
- **Thread Safety:** These systems are designed to be run by a single-threaded, or at least phase-locked, ECS update loop. The use of a CommandBuffer for all mutations (like removing a component) is a critical pattern here. It ensures that state changes are not applied immediately but are queued. These queued commands are then executed in a deterministic, synchronized phase of the game loop, preventing race conditions and ensuring data consistency. Direct, multi-threaded invocation of these systems would be catastrophic.

## API Surface
The public methods on these systems are callbacks, not intended for direct invocation. They are part of a contract with the ECS engine, which calls them when their subscribed queries are matched.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OnAdd.onEntityAdded | void | O(1) | **Callback.** Triggered when an NPCMountComponent is added to an entity. Manages mount setup and network packet dispatch. |
| DismountOnMountDeath.onComponentAdded | void | O(1) | **Callback.** Triggered when a DeathComponent is added to a mounted NPC. Resets the player's movement state. |
| DismountOnPlayerDeath.onComponentAdded | void | O(1) | **Callback.** Triggered when a DeathComponent is added to a player. Initiates the dismount process for their current mount, if any. |

## Integration Patterns

### Standard Usage
Developers do not interact with these systems directly. The systems are automatically registered with the engine. The standard pattern for initiating the mount process is to add the NPCMountComponent to an NPC entity via a CommandBuffer. This action implicitly triggers the OnAdd system.

```java
// Correctly initiating a mount sequence
// This code would exist in a higher-level interaction system.

// Assume 'playerRef' and 'npcRef' are valid entity references
// Assume 'commandBuffer' is provided by the current system's update method

NPCMountComponent mount = new NPCMountComponent(playerRef, originalRoleIndex);
commandBuffer.addComponent(npcRef, mount);

// The NPCMountSystems.OnAdd system will now automatically be invoked by the engine
// in a future processing phase.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCMountSystems.OnAdd()`. The engine's System Manager is solely responsible for the lifecycle of these systems.
- **Direct Invocation:** Never call the `onEntityAdded` or `onComponentAdded` methods directly. This bypasses the ECS engine's query and command buffer mechanisms, leading to race conditions, state corruption, and server instability.
- **Manual State Management:** Do not attempt to replicate the logic of these systems by manually sending MountNPC packets or removing the NPCMountComponent. This will result in desynchronization between the server's state and the client's view. Always trigger the process by adding the component.

## Data Pipeline
The data flow for these systems is event-based, originating from component state changes within the ECS.

> **Flow: Mount Initiation**
> Gameplay Event (e.g., Player Interaction) -> Interaction System adds **NPCMountComponent** -> ECS Engine detects change -> **NPCMountSystems.OnAdd** -> Reads Player and NPC data -> Sends **MountNPC Packet** to Client & Updates **Player Component** state.

> **Flow: Dismount on Death**
> Damage System adds **DeathComponent** to Player or Mount -> ECS Engine detects change -> **DismountOnPlayerDeath** or **DismountOnMountDeath** -> Executes cleanup logic via **MountPlugin** and **CommandBuffer**.


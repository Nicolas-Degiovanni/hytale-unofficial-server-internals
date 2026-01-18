---
description: Architectural reference for EnterBedSystem
---

# EnterBedSystem

**Package:** com.hypixel.hytale.builtin.beds.sleep.systems.player
**Type:** ECS System

## Definition
```java
// Signature
public class EnterBedSystem extends RefChangeSystem<EntityStore, MountedComponent> {
```

## Architecture & Concepts
The EnterBedSystem is a server-side, reactive system within the Entity Component System (ECS) framework. Its primary function is to validate and provide player feedback when a player entity attempts to sleep in a bed.

This system operates by listening for changes to the MountedComponent on entities that also possess a PlayerRef component. When a MountedComponent is added or updated, the system inspects it. If the mount type is identified as a bed (BlockMountType.Bed), it triggers the core sleep validation logic.

Architecturally, this class decouples the generic act of "mounting" an entity from the specific game rules of "sleeping". The MountedComponent is a general-purpose flag, but this system imbues it with context-specific behavior for beds. It queries the world's state via the CanSleepInWorld utility to determine if conditions are suitable for sleeping (e.g., time of day, game rules). If sleeping is not permitted, the system is responsible for constructing and sending a localized, user-friendly message to the player explaining the reason.

## Lifecycle & Ownership
-   **Creation:** A single instance of EnterBedSystem is instantiated by the server's core ECS engine during the bootstrap process. Systems are typically discovered via reflection or a predefined registry and are not created manually.
-   **Scope:** The system is a stateless singleton that persists for the entire lifetime of the server. It is not tied to a specific world or player session; it processes component changes for all relevant entities across the server.
-   **Destruction:** The instance is destroyed and garbage collected only when the server process shuts down.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no mutable instance fields and all data required for its operations is provided as arguments to its methods by the ECS framework. This design ensures that each invocation is idempotent and free from side effects caused by previous executions.
-   **Thread Safety:** This system is **not thread-safe** and is not designed for concurrent access. The Hytale ECS framework guarantees that all system lifecycle methods (onComponentAdded, onComponentSet, etc.) are invoked from a single, designated game loop thread for a given world tick. All interactions with the Store and CommandBuffer are therefore implicitly synchronized by the engine.

## API Surface
The public API is primarily defined by its parent, RefChangeSystem, and is invoked exclusively by the ECS engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(ref, component, store, commandBuffer) | void | O(1) | Engine callback. Invoked when a MountedComponent is added to an entity matching the QUERY. |
| onComponentSet(ref, old, new, store, commandBuffer) | void | O(1) | Engine callback. Invoked when a MountedComponent is modified on an entity matching the QUERY. |
| check(ref, component, store) | void | O(1) | Internal logic to filter for bed mounts and trigger the onEnterBed flow. |
| onEnterBed(ref, store) | void | O(C) | Core logic. Checks world sleep conditions and sends feedback messages to the player. Complexity depends on the world state check. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. The system is triggered implicitly by the game engine when another part of the codebase adds or modifies a MountedComponent on a player entity.

The typical trigger flow originates from player-block interaction logic:
```java
// Example of code that would TRIGGER this system
// This code would exist elsewhere, e.g., in a bed interaction handler.

// Get a command buffer to queue entity changes
CommandBuffer<EntityStore> commands = ...;
Ref<EntityStore> playerEntityRef = ...;

// Adding this component causes the ECS engine to invoke EnterBedSystem
MountedComponent bedMount = new MountedComponent(targetBed, BlockMountType.Bed);
commands.setComponent(playerEntityRef, bedMount);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new EnterBedSystem()`. The ECS engine is solely responsible for the lifecycle of systems. Direct instantiation will result in a non-functional object that is not registered to receive engine callbacks.
-   **Manual Invocation:** Do not call methods like onComponentAdded or check directly. This bypasses the ECS framework's transactional nature and can lead to state corruption, race conditions, or missed component updates. All system interaction must occur via changes to components in a Store or CommandBuffer.

## Data Pipeline
The system acts as a processor in a larger data flow that begins with player input and ends with a UI update on the client.

> Flow:
> Player Interaction with Bed Block -> Server adds **MountedComponent** to Player Entity -> ECS Engine dispatches component change event -> **EnterBedSystem.onComponentAdded** -> System queries **CanSleepInWorld** rules -> System sends **Message** to PlayerRef -> Network layer serializes and sends feedback packet -> Client UI renders translated message.


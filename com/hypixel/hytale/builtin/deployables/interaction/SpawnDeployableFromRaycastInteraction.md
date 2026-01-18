---
description: Architectural reference for SpawnDeployableFromRaycastInteraction
---

# SpawnDeployableFromRaycastInteraction

**Package:** com.hypixel.hytale.builtin.deployables.interaction
**Type:** Transient / Configuration Object

## Definition
```java
// Signature
public class SpawnDeployableFromRaycastInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The SpawnDeployableFromRaycastInteraction is a server-side implementation of the core interaction system, designed to handle the placement of deployable entities into the world. It is a data-driven class, meaning its specific behavior—such as the entity to spawn, placement constraints, and resource costs—is defined in external asset files, not hardcoded.

Its primary architectural role is to act as the server-side authority for an action initiated by a client. It consumes synchronized client-side state, specifically the results of a client-world raycast, to determine a valid placement location. It then validates this action against game rules (e.g., placement distance, surface angle, resource costs) and, if successful, queues the entity creation command into the world's CommandBuffer.

This class is a concrete example of the Command pattern used heavily within the engine. It encapsulates all the logic for a specific "spawn from raycast" action, but it does not execute the world modification directly. Instead, it submits a command for deferred execution by the main game loop, ensuring deterministic and thread-safe world state changes.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the new keyword. They are deserialized and instantiated by the engine's asset loading system via the static **CODEC** field. Each instance represents a single, reusable interaction configuration defined in an asset file (e.g., a JSON file for a placeable item).
-   **Scope:** An instance persists for the lifetime of the asset registry. It is effectively a stateless singleton relative to its configuration, shared by all players and entities that trigger this specific interaction type.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down, a world is unloaded, or assets are reloaded, at which point the defining configuration is no longer in memory.

## Internal State & Concurrency
-   **State:** The internal state of this class (config, entityStats, maxPlacementDistance) is configured once at load time during codec deserialization. The **afterDecode** hook finalizes this state by resolving stat names into more performant integer IDs. After this initial setup, the object's state is effectively immutable. It does not cache or modify its own state during interaction execution.
-   **Thread Safety:** This class is inherently thread-safe. Its methods operate exclusively on data passed in through the InteractionContext and ComponentAccessor arguments, which are scoped to a single interaction event. All world-modifying actions are funneled through the provided CommandBuffer, which is the engine's designated mechanism for synchronizing state changes from multiple threads into the main game loop. A single instance of this class can be safely invoked by multiple threads processing different interactions concurrently.

## API Surface
The public contract is primarily defined by its parent class, SimpleInstantInteraction. The key overrides dictate its behavior within the interaction framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldown) | void | O(N) | The primary entry point for executing the interaction. Validates client data and game rules, then queues a spawn command. Complexity is O(N) where N is the number of stat costs to check. |
| canAfford(entityRef, accessor) | boolean | O(N) | Checks if the interacting entity possesses the required resources (EntityStats) defined in the configuration. |
| needsRemoteSync() | boolean | O(1) | Returns **true**, signaling to the interaction system that this server-side logic cannot proceed without data from the client. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns **WaitForDataFrom.Client**, specifying that the required synchronization data must originate from the game client. |
| configurePacket(packet) | void | O(1) | Populates a network packet with configuration data (e.g., costs, max distance) for the client. This enables client-side prediction and UI feedback. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. Instead, it is specified as the handler for an interaction within an asset definition file. The engine's interaction module resolves this definition at runtime.

**Conceptual Asset Definition (JSON)**
```json
{
  "interaction": {
    "type": "SpawnDeployableFromRaycastInteraction",
    "Config": {
      "asset": "mygame:deployables/sentry_turret"
    },
    "MaxPlacementDistance": 10.0,
    "PreviewStatConditions": {
      "mygame:stats/iron_ingots": 50.0,
      "mygame:stats/power_cores": 1.0
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SpawnDeployableFromRaycastInteraction()`. This bypasses the critical CODEC-based deserialization, resulting in a null and unconfigured object that will cause NullPointerExceptions.
-   **State Mutation:** Do not attempt to modify the public fields like config or maxPlacementDistance after the object has been initialized by the asset loader. This would violate its stateless contract and affect every interaction of this type across the entire server, leading to unpredictable behavior.
-   **Server-Side Raycasting:** Do not attempt to perform a new raycast within this class. The design pattern requires that the client's raycast is the source of truth for the interaction's target location. This prevents client-server desynchronization where the server and client disagree on what the player is looking at.

## Data Pipeline
The flow of data for this interaction is a clear example of a client-authoritative input model with server-side validation.

> Flow:
> 1.  **Client:** Player input (e.g., Right Mouse Button) triggers an item interaction.
> 2.  **Client:** The client-side interaction system performs a raycast against the world geometry.
> 3.  **Client:** An Interaction packet is sent to the server. This packet's payload includes an InteractionSyncData object containing the raycast hit position, surface normal, distance, and the player's current rotation.
> 4.  **Server:** The network layer routes the packet to the server's InteractionModule.
> 5.  **Server:** The module retrieves the configured **SpawnDeployableFromRaycastInteraction** instance and invokes its `firstRun` method, providing an InteractionContext that wraps the client's InteractionSyncData.
> 6.  **Server (firstRun):** The method validates the raycast distance against `maxPlacementDistance`, checks the surface normal if `allowPlaceOnWalls` is false, and calls `canAfford` to verify entity stats.
> 7.  **Server (firstRun):** If all checks pass, it constructs a spawn command and enqueues it on the CommandBuffer.
> 8.  **Server (Game Loop):** The CommandBuffer is processed at the end of the tick, and the deployable entity is formally spawned into the world state.


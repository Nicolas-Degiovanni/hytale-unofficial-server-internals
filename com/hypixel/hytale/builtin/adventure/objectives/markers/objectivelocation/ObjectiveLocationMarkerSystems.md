---
description: Architectural reference for ObjectiveLocationMarkerSystems
---

# ObjectiveLocationMarkerSystems

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers.objectivelocation
**Type:** Utility / Container

## Definition
```java
// Signature
public class ObjectiveLocationMarkerSystems {
    // Contains nested static System classes
}
```

## Architecture & Concepts
The ObjectiveLocationMarkerSystems class is not a concrete system itself but serves as a static container and namespace for a suite of highly-specialized Entity Component Systems (ECS). These systems collectively manage the entire lifecycle of an Objective Location Marker entity, from its initial creation and data hydration to its runtime player tracking and eventual destruction.

This class embodies a strong separation of concerns pattern within the ECS framework:
- **EnsureNetworkSendableSystem:** Handles network prerequisite setup.
- **InitSystem:** Manages entity initialization and teardown, acting as a bridge between the entity's existence and the persistent state of its associated Objective.
- **TickingSystem:** Implements the core, stateful runtime logic that activates objectives and tracks player interaction.

By encapsulating these related systems within a single outer class, the engine maintains organizational clarity for one of the adventure mode's most critical gameplay mechanics.

---
description: Architectural reference for ObjectiveLocationMarkerSystems.EnsureNetworkSendableSystem
---

# ObjectiveLocationMarkerSystems.EnsureNetworkSendableSystem

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers.objectivelocation
**Type:** Utility System

## Definition
```java
// Signature
public static class EnsureNetworkSendableSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This system serves a singular, critical purpose: to guarantee that any entity functioning as an Objective Location Marker is addressable over the network. It operates on a simple query, identifying entities that possess an ObjectiveLocationMarker component but lack a NetworkId component.

Its role is that of a data integrity and prerequisite fulfillment system. By immediately adding a NetworkId upon entity creation, it ensures that higher-level systems, particularly the TickingSystem which sends network packets to clients, can operate without needing to perform redundant checks or handle networking edge cases. It is a foundational layer that makes the rest of the marker's logic possible.

## Lifecycle & Ownership
- **Creation:** Instantiated by the ECS scheduler during system registration at server startup.
- **Scope:** Lives for the entire duration of the server session. As a stateless system, a single instance serves all worlds.
- **Destruction:** Decommissioned during server shutdown.

## Internal State & Concurrency
- **State:** This system is entirely stateless. It does not store or cache any data between invocations.
- **Thread Safety:** The system is inherently thread-safe due to its stateless nature. It operates on entities provided by the ECS scheduler, and its actions via the Holder are managed by the engine's concurrency model.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback triggered by the ECS scheduler when a new entity matches the query. Assigns a new NetworkId to the entity. |

## Integration Patterns

### Standard Usage
This system is not intended for direct developer interaction. It is automatically registered and executed by the server's core ECS engine. Its operation is transparent to other game logic.

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call `onEntityAdd` directly. The ECS scheduler is solely responsible for its execution.
- **Dependency:** Do not create systems that depend on this system completing. Its execution order relative to other `onEntityAdd` systems is not guaranteed beyond the standard ECS scheduling phases.

---
description: Architectural reference for ObjectiveLocationMarkerSystems.InitSystem
---

# ObjectiveLocationMarkerSystems.InitSystem

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers.objectivelocation
**Type:** Lifecycle Management System

## Definition
```java
// Signature
public static class InitSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The InitSystem is the primary lifecycle manager for Objective Location Marker entities. It acts as the critical bridge between an entity's raw component data and its fully realized, stateful representation in the game world. It is responsible for two opposing but symmetrical flows: data hydration on creation and data persistence on destruction.

- **On Creation (`onEntityAdded`):** When a marker entity is created, this system is responsible for loading its corresponding ObjectiveLocationMarkerAsset. It validates the asset, loads any pre-existing Objective state from the ObjectiveDataStore if an `activeObjectiveUUID` is present, and configures other components on the entity, such as updating the ModelComponent's collision box to match the objective's defined area. If any part of this hydration fails, the system will command the entity to be removed to prevent a partially-initialized, non-functional marker from existing in the world.

- **On Destruction (`onEntityRemove`):** When a marker entity is removed, this system intercepts the event to perform critical cleanup. It serializes the current state of the active Objective and saves it to disk via the ObjectiveDataStore. This ensures that quest progress is not lost if the entity is unloaded (e.g., chunk unloading). It then formally unloads the Objective from memory.

## Lifecycle & Ownership
- **Creation:** Instantiated by the ECS scheduler during system registration.
- **Scope:** Persists for the entire server session.
- **Destruction:** Decommissioned during server shutdown.

## Internal State & Concurrency
- **State:** This system is stateless. All state it manipulates is stored either on the entity's components (e.g., ObjectiveLocationMarker) or in external services (e.g., ObjectiveDataStore, ObjectiveLocationMarkerAsset map).
- **Thread Safety:** The system's operations are executed within the context of the ECS scheduler's command buffer, ensuring that component modifications are queued and applied in a thread-safe manner. Direct access to external services like ObjectivePlugin must be thread-safe.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(N) | Hydrates the entity. Complexity depends on asset loading and objective deserialization. Can issue a command to remove the entity on failure. |
| onEntityRemove(...) | void | O(N) | Persists the active objective's state to disk. Complexity depends on objective serialization. |

## Integration Patterns

### Standard Usage
This system is fully automated by the ECS engine. A designer or developer creates an entity with the ObjectiveLocationMarker component, and this system will automatically handle its setup and teardown.

### Anti-Patterns (Do NOT do this)
- **State Assumption:** Do not assume an entity is fully initialized in another system that runs in the same tick as its creation. This system's `onEntityAdded` logic may not have completed yet.
- **Manual Cleanup:** Do not attempt to manually save objective data when removing a marker entity. This system is the sole authority for this process and duplicating it will lead to race conditions and data corruption.

## Data Pipeline
> Flow (Creation):
> Entity Added -> **InitSystem.onEntityAdded** -> ObjectiveLocationMarkerAsset.getAssetMap -> ObjectiveDataStore.loadObjective -> CommandBuffer.putComponent -> Fully Hydrated Entity

> Flow (Destruction):
> Entity Removed -> **InitSystem.onEntityRemove** -> ObjectiveDataStore.saveToDisk -> ObjectiveDataStore.unloadObjective

---
description: Architectural reference for ObjectiveLocationMarkerSystems.TickingSystem
---

# ObjectiveLocationMarkerSystems.TickingSystem

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers.objectivelocation
**Type:** Core Logic System

## Definition
```java
// Signature
public static class TickingSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The TickingSystem is the runtime heart of the objective marker. It executes every tick for all active markers and is responsible for detecting and managing player interactions. Its logic can be modeled as a state machine for each marker entity.

1.  **Inactive State:** If a marker has no active objective, the system continuously checks for players within its trigger area and verifies that all `triggerConditions` are met. Once conditions are satisfied, it transitions the marker to the Active state by creating a new Objective instance via the `ObjectiveLocationMarkerAsset`. This is the activation phase.

2.  **Active State:** Once an objective is active, the system's focus shifts to player tracking. It uses efficient spatial queries to get all players within the marker's broader `exitArea`.
    - **Player Entry:** For players inside the `entryArea` who are not yet tracked, it adds them to the objective's set of active players and sends a `TrackOrUpdateObjective` packet to their client, making the objective visible on their UI.
    - **Player Exit:** For players who were previously tracked but have now left the `exitArea`, it removes them from the active set and sends an `UntrackObjective` packet.
    - This continuous tracking ensures that only nearby, relevant players are participating in the objective, saving both server and client resources.

3.  **Completed State:** If the system detects that the active objective has been completed, it issues a command to remove the marker entity from the world, triggering the `InitSystem`'s cleanup logic. This facilitates a "self-destruct" pattern for single-use objectives.

This system has a declared dependency on PlayerSpatialSystem, ensuring that player location data is fully updated before this system attempts to query it.

## Lifecycle & Ownership
- **Creation:** Instantiated by the ECS scheduler at server startup.
- **Scope:** A single instance persists for the entire server session.
- **Destruction:** Decommissioned during server shutdown.

## Internal State & Concurrency
- **State:** The system itself is stateless. All state is read from and written to the ObjectiveLocationMarker component on each entity and the associated Objective object it holds.
- **Thread Safety:** The system explicitly opts out of parallel execution by returning `false` from `isParallel`. This is a critical design choice, likely because the logic for activating objectives and managing player lists is sensitive and difficult to parallelize without complex locking. All operations on a given tick occur serially.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(P) | The core logic loop executed for each marker entity per tick. Complexity is proportional to the number of players (P) within the marker's spatial query range. |

## Integration Patterns

### Standard Usage
This system is fully automated. Its logic is driven by player movement and interaction with the world. Developers influence its behavior by configuring the `ObjectiveLocationMarkerAsset` and the components on the marker entity.

### Anti-Patterns (Do NOT do this)
- **Performance Overload:** Do not place an excessive number of markers with very large trigger areas in a small region. While the spatial queries are efficient, a high density of overlapping, ticking markers can still create CPU load.
- **Ignoring Dependencies:** Attempting to replicate this system's logic without correctly ordering it after `PlayerSpatialSystem` will result in severe race conditions, where player location checks are performed using data from the previous tick.

## Data Pipeline
> Flow (Player Enters Area):
> Player Position Update -> PlayerSpatialSystem -> **TickingSystem.tick** -> SpatialResource Query -> Active Player Set Update -> TrackOrUpdateObjective Packet -> Client Network Handler -> Client UI Update


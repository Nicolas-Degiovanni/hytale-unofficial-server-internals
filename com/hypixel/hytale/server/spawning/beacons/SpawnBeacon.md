---
description: Architectural reference for SpawnBeacon
---

# SpawnBeacon

**Package:** com.hypixel.hytale.server.spawning.beacons
**Type:** Stateful Entity

## Definition
```java
// Signature
public class SpawnBeacon extends Entity {
```

## Architecture & Concepts

The SpawnBeacon is a specialized, non-physical entity that acts as a persistent, location-based spawner within the game world. Unlike transient spawn commands or global spawn systems, a SpawnBeacon is a concrete entity with a transform, state, and a defined lifecycle, allowing for precise placement and configuration by designers or procedural generation systems.

Architecturally, it serves as a bridge between static asset configuration (the spawn rules defined in a BeaconNPCSpawn asset) and the dynamic Entity-Component-System (ECS) world. It is not an autonomous system that runs on every tick; rather, it is a stateful object that is acted upon by external systems, such as SpawnBeaconSystems.

Its core responsibility is to orchestrate the spawning of one or more NPCs when triggered. This process involves several key collaborators:

*   **BeaconSpawnWrapper:** A data-holder that provides the beacon with its configuration, such as which NPC roles can be spawned and in what quantities.
*   **FloodFillPositionSelector:** A required sibling component that performs the computationally expensive task of finding valid, non-obstructed spawn locations within the beacon's vicinity. The SpawnBeacon delegates all position-finding logic to this component.
*   **NPCPlugin and FlockPlugin:** Centralized services used to perform the actual instantiation of NPC and flock entities into the world.

A key design feature is its self-culling mechanism. If a beacon repeatedly fails to find valid spawn locations for all of its configured NPC roles, it will automatically remove itself from the world to prevent performance degradation from repeated failed spawn attempts.

## Lifecycle & Ownership

-   **Creation:** A SpawnBeacon is instantiated as part of world data deserialization via its registered CODEC. It is intended to be placed in the world by developers or procedural systems, not created dynamically during gameplay logic. Its configuration, linked by the spawnConfigId, is resolved and attached post-instantiation.

-   **Scope:** The entity persists for the entire lifetime of the world chunk it occupies. Its state, including the spawnConfigId, is saved and loaded across server sessions.

-   **Destruction:** A SpawnBeacon is destroyed under two conditions:
    1.  Explicitly removed by an administrative command or world editing tool.
    2.  **Automatic Self-Destruction:** The beacon will call its own remove method if it determines that all possible NPC roles defined in its configuration are unspawnable. This is tracked internally via the unspawnableRoles set and prevents orphaned, non-functional beacons from accumulating in the world.

## Internal State & Concurrency

-   **State:** The SpawnBeacon is highly stateful and mutable. Its primary internal state is the unspawnableRoles set, which tracks which NPC types have failed to spawn. This state is critical for its self-destruction logic. The state is reset if the beacon is moved, as a new location may have different spawning constraints.

-   **Thread Safety:** This class is **not thread-safe**. As a subclass of Entity, all interactions with a SpawnBeacon instance must occur on the main server thread that owns the corresponding World. The use of ThreadLocalRandom in the manualTrigger method further reinforces this single-threaded design assumption. Unsynchronized access from other threads will lead to state corruption and severe concurrency violations within the ECS framework.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setSpawnWrapper(wrapper) | void | O(1) | Binds a spawn configuration to the beacon. This is a critical initialization step. |
| manualTrigger(ref, selector, targetRef, store) | boolean | O(N * M) | Attempts to perform a spawn cycle. N is the number of concurrent spawns, M is the complexity of position selection. Returns true if at least one entity was spawned. |
| moveTo(ref, x, y, z, accessor) | void | O(K) | Overrides base entity behavior. Resets the beacon's spawn failure state and triggers a rebuild of the position cache (K). |

## Integration Patterns

### Standard Usage

A SpawnBeacon should never be instantiated or triggered directly. It is designed to be managed by a dedicated system that queries for beacon entities and triggers them based on game logic, such as player proximity or a timed interval.

```java
// Executed within a server system (e.g., SpawnBeaconSystems)
Store<EntityStore> store = world.getModule(EntityModule.class).getStore();

// Query for all SpawnBeacon entities near a player
for (Ref<EntityStore> beaconRef : findBeaconsNearPlayer(playerRef)) {
    SpawnBeacon beacon = store.getComponent(beaconRef, SpawnBeacon.getComponentType());
    FloodFillPositionSelector selector = store.getComponent(beaconRef, FloodFillPositionSelector.getComponentType());

    // The system is responsible for checking conditions and triggering
    if (beacon != null && selector != null && shouldTrigger(beacon)) {
        beacon.manualTrigger(beaconRef, selector, playerRef, store);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new SpawnBeacon()`. Entities must be created via the `World` or `EntityStore` to be correctly registered and managed by the engine.
-   **Missing Dependencies:** A SpawnBeacon entity is non-functional without a sibling `FloodFillPositionSelector` component. The code contains an assertion for this, but failure to provide it will result in a crash.
-   **External State Modification:** Do not modify the `unspawnableRoles` set from outside the class. This internal state is critical to the beacon's self-destruction lifecycle and should not be tampered with.

## Data Pipeline

The data flow for a successful spawn operation originates from an external system and terminates with a new entity being added to the world and scheduled for ticking.

> Flow:
> External Trigger (e.g., Player Proximity System) -> **SpawnBeacon.manualTrigger()** -> Role Selection & Validation -> FloodFillPositionSelector.prepareSpawnContext() -> Position & Rotation Calculation -> NPCPlugin.spawnEntity() -> EntityStore Update -> **postSpawn()** callback -> NewSpawnStartTickingSystem.queueNewSpawn()


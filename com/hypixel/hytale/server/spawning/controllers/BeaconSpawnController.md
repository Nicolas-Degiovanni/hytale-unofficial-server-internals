---
description: Architectural reference for BeaconSpawnController
---

# BeaconSpawnController

**Package:** com.hypixel.hytale.server.spawning.controllers
**Type:** Transient

## Definition
```java
// Signature
public class BeaconSpawnController extends SpawnController<NPCBeaconSpawnJob> {
```

## Architecture & Concepts

The BeaconSpawnController is a stateful, server-side class responsible for managing the complete NPC spawning lifecycle for a single spawn beacon entity. It acts as the dedicated logic engine for a `LegacySpawnBeaconEntity` component, translating its configuration data into concrete spawning behavior within the game world.

This controller is a specialized implementation of the more generic `SpawnController` base class. Its primary responsibilities include:

*   **Population Management:** Enforcing limits on the total number of active NPCs and the number of concurrent spawning operations originating from its associated beacon.
*   **Target Selection:** Identifying and cycling through nearby players to use as anchor points for new spawn attempts. It includes a threat-based sorting mechanism to prioritize spawning near players with fewer associated entities.
*   **Job Creation:** Instantiating and configuring `NPCBeaconSpawnJob` instances. These jobs encapsulate the asynchronous and potentially expensive process of finding a valid spawn location in the world geometry.
*   **State Tracking:** Maintaining an internal state of all spawned entities, active players in the region, and cooldowns. It directly manages the list of living NPCs that "belong" to its beacon.
*   **Despawning Logic:** Tracking idle timers for spawned NPCs and managing the despawn conditions for the beacon entity itself if no players are nearby.

It operates as a bridge between the static configuration defined in spawn assets (`BeaconNPCSpawn`) and the dynamic, runtime state of the world.

### Lifecycle & Ownership

*   **Creation:** A BeaconSpawnController is instantiated by the server's spawning system when a `LegacySpawnBeaconEntity` component becomes active in the world. Its lifecycle is directly and exclusively tied to its owner entity. The `ownerRef` field establishes this permanent link.
*   **Scope:** The instance persists as long as its corresponding beacon entity exists and is active within a loaded world region. It is not shared between beacons; each beacon has its own dedicated controller instance.
*   **Destruction:** The object is eligible for garbage collection when the owning `LegacySpawnBeaconEntity` is removed from the world. There is no explicit destruction or cleanup method; its state is lost when the object is garbage collected.

## Internal State & Concurrency

*   **State:** This class is highly mutable and stateful. It maintains several collections that track the dynamic state of its spawning region, including lists of spawned entities (`spawnedEntities`), nearby players (`playersInRegion`), and maps for tracking per-player entity counts and despawn timers. Configuration values are cached from a `BeaconSpawnWrapper` during the `initialise` call.

*   **Thread Safety:** **This class is not thread-safe.** Its internal collections (e.g., `ObjectArrayList`, `Object2IntOpenHashMap`) are not concurrent. All method calls that modify internal state must be performed from a single, synchronized context, which is assumed to be the main server thread for the world region in which the beacon resides. Unmanaged multi-threaded access will lead to `ConcurrentModificationException` and severe state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialise(BeaconSpawnWrapper) | void | O(1) | Caches spawning parameters from the asset wrapper. Must be called before the controller can create jobs. |
| createRandomSpawnJob(ComponentAccessor) | NPCBeaconSpawnJob | O(1) | The primary work-horse method. Attempts to create a spawn job if all conditions (spawn caps, player proximity) are met. Returns null if a job cannot be created. |
| notifySpawnedEntityExists(Ref, ComponentAccessor) | void | O(1) | Registers a newly created NPC with this controller, adding it to the internal tracking list. |
| notifyNPCRemoval(Ref, ComponentAccessor) | void | O(N) | Deregisters an NPC that has been removed from the world. Involves a linear scan of the `spawnedEntities` list. |
| onJobFinished(ComponentAccessor) | void | O(1) | Callback invoked by the spawning system when a job successfully completes. |
| hasSlots() | boolean | O(1) | Checks if the current number of spawned entities is below the maximum configured limit. |
| markNPCUnspawnable(int) | void | O(1) | Adds an NPC role to a temporary blacklist, preventing it from being selected for spawning. |

## Integration Patterns

### Standard Usage

The BeaconSpawnController is not intended to be used directly by most game logic. It is managed by a higher-level `SpawningSystem`. This system is responsible for ticking the controller, updating it with nearby players, and dispatching the jobs it creates.

```java
// Within a hypothetical SpawningSystem update loop
ComponentAccessor<EntityStore> accessor = world.getComponentAccessor(EntityStore.class);
BeaconSpawnController controller = getControllerForBeacon(beaconRef);

// The system first updates the controller's view of the world
updatePlayersInRegion(controller);
updateScaledSpawnCounts(controller);

// Then, it attempts to generate new work from the controller
for (int i = 0; i < BeaconSpawnController.MAX_ATTEMPTS_PER_TICK; i++) {
    NPCBeaconSpawnJob job = controller.createRandomSpawnJob(accessor);
    if (job != null) {
        jobSystem.submit(job);
    } else {
        // No more jobs can be created this tick
        break;
    }
}
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not use `new BeaconSpawnController()`. The lifecycle is managed by the entity-component and spawning systems. Manually creating a controller will result in a disconnected, non-functional object.
*   **State Tampering:** Do not directly modify the collections returned by getters like `getSpawnedEntities()`. For example, calling `controller.getSpawnedEntities().add(someRef)` will corrupt the controller's internal accounting and break its population management. Use the provided `notify` methods instead.
*   **Multi-threaded Access:** Do not call methods on a controller instance from multiple threads. All interactions must be serialized through the main world tick to prevent race conditions.

## Data Pipeline

The controller facilitates a flow of logic that begins with a server tick and ends with a new NPC entity in the world.

> Flow:
> Server Tick -> Spawning System -> **BeaconSpawnController**.createRandomSpawnJob() -> NPCBeaconSpawnJob -> Job System Execution -> Entity Creation -> **BeaconSpawnController**.notifySpawnedEntityExists() -> Internal State Update


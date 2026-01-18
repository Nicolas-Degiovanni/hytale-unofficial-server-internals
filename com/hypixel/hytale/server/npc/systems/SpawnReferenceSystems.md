---
description: Architectural reference for SpawnReferenceSystems
---

# SpawnReferenceSystems

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Utility / Namespace

## Architecture & Concepts
The SpawnReferenceSystems class is a static container for a suite of specialized systems within the Hytale Entity Component System (ECS) framework. It does not represent a single object or service but rather groups together the logic responsible for managing the lifecycle and relationship between spawned Non-Player Characters (NPCs) and their originating spawn points, which can be either a Spawn Beacon or a Spawn Marker.

These systems are critical for server stability and correctness, ensuring that:
1.  NPCs remain correctly associated with their spawner.
2.  Spawner population counts are accurately maintained.
3.  Orphaned NPCs (whose spawners have been removed or unloaded) are gracefully cleaned up from the world.

Each nested class targets a specific event (add/remove, per-tick update) and a specific spawner type (Beacon or Marker), creating a clear separation of concerns.

---

## BeaconAddRemoveSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Managed System (ECS RefSystem)

### Definition
```java
// Signature
public static class BeaconAddRemoveSystem extends RefSystem<EntityStore> {
```

### Architecture & Concepts
BeaconAddRemoveSystem is an event-driven system that manages the bidirectional relationship between an NPCEntity and its LegacySpawnBeaconEntity. It operates by listening for the addition and removal of entities that match its query—specifically, entities possessing both a SpawnBeaconReference and an NPCEntity component.

Its primary role is to act as a transactional link between the NPC and the spawner's state. When an NPC is loaded into the world, this system validates the link to its beacon, confirms the beacon has available spawn slots, and notifies the beacon's BeaconSpawnController of the NPC's existence. Conversely, when the NPC is removed, this system informs the controller, freeing up the spawn slot for future use. This prevents over-spawning and ensures spawner population caps are respected.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central ECS System Registry during server initialization. Its dependencies (ComponentType instances) are injected via its constructor.
-   **Scope:** Singleton for the duration of the server session. It is a stateless processor that exists as long as the server is running.
-   **Destruction:** Decommissioned and garbage collected during server shutdown when the System Registry is cleared.

### Internal State & Concurrency
-   **State:** This system is entirely stateless. It holds no mutable fields and relies solely on the component data of the entities it processes, which are provided by the ECS framework through method parameters.
-   **Thread Safety:** Operations within this system are not designed to be called from arbitrary threads. The ECS framework guarantees that the onEntityAdded and onEntityRemove callbacks are executed in a controlled, serialized manner as part of entity lifecycle management. All world mutations are deferred through the CommandBuffer, ensuring thread-safe application at the end of the current processing step.

### API Surface
The public contract is defined by the RefSystem base class. These methods are callbacks invoked by the ECS framework and should not be called directly by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(1) | Callback triggered when a matching NPC is added. Re-establishes the spawner link and updates the spawn controller. |
| onEntityRemove(...) | void | O(1) | Callback triggered when a matching NPC is removed. Notifies the spawn controller to release the occupied slot. |

### Integration Patterns

#### Standard Usage
This system is not used directly. It is automatically registered and invoked by the server's ECS engine. Its functionality is implicitly triggered by creating, loading, or destroying NPCs that originate from a LegacySpawnBeaconEntity.

#### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using new BeaconAddRemoveSystem(). The ECS framework manages its lifecycle.
-   **Manual Invocation:** Do not call onEntityAdded or onEntityRemove directly. Doing so would bypass the ECS framework's state management and lead to world corruption.

### Data Pipeline
> **Add Flow:**
> Entity Load/Spawn -> ECS Framework -> **BeaconAddRemoveSystem.onEntityAdded** -> Component Data Access -> BeaconSpawnController.notifySpawnedEntityExists

> **Remove Flow:**
> Entity Despawn/Unload -> ECS Framework -> **BeaconAddRemoveSystem.onEntityRemove** -> Component Data Access -> BeaconSpawnController.notifyNPCRemoval

---

## MarkerAddRemoveSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Managed System (ECS RefSystem)

### Definition
```java
// Signature
public static class MarkerAddRemoveSystem extends RefSystem<EntityStore> {
```

### Architecture & Concepts
Analogous to the BeaconAddRemoveSystem, this system manages the relationship between an NPCEntity and its SpawnMarkerEntity. It listens for the addition and removal of NPCs that are linked to a spawn marker via the SpawnMarkerReference component.

Upon NPC addition (e.g., world chunk load), it re-establishes the connection, refreshes timeout counters on both the NPC and the marker, and logs the synchronization. Upon NPC removal, its critical responsibility is to decrement the spawn count on the SpawnMarkerEntity. If the spawn count drops to zero and the marker is not configured for real-time respawning, this system is responsible for setting the marker's future respawn time.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS System Registry during server startup.
-   **Scope:** Singleton for the duration of the server session.
-   **Destruction:** Decommissioned during server shutdown.

### Internal State & Concurrency
-   **State:** Stateless. All operations are performed on component data passed in by the ECS framework.
-   **Thread Safety:** Not thread-safe for external calls. The ECS framework ensures safe execution via its single-threaded, sequential processing of entity add/remove events. World modifications are safely queued in the CommandBuffer.

### API Surface
The public contract is defined by the RefSystem base class and is invoked exclusively by the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(1) | Callback triggered when a marker-spawned NPC is added. Refreshes state and logs the link. |
| onEntityRemove(...) | void | O(N) | Callback triggered when a marker-spawned NPC is removed. Decrements the marker's spawn count and may initiate a respawn cooldown. Complexity is O(N) where N is the number of NPCs linked to the marker, due to array filtering. |

### Integration Patterns

#### Standard Usage
This system is fully managed by the ECS framework. Its logic is applied automatically whenever an NPC with a SpawnMarkerReference is added to or removed from the world.

#### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use new MarkerAddRemoveSystem(). The system's lifecycle is controlled by the server.
-   **Bypassing the System:** Manually removing an NPC without allowing this system to run (e.g., by only removing its components) will cause the SpawnMarkerEntity to retain an incorrect spawn count, potentially preventing future spawns.

### Data Pipeline
> **Add Flow:**
> Entity Load/Spawn -> ECS Framework -> **MarkerAddRemoveSystem.onEntityAdded** -> CommandBuffer -> SpawnMarkerEntity State Update

> **Remove Flow:**
> Entity Despawn/Unload -> ECS Framework -> **MarkerAddRemoveSystem.onEntityRemove** -> Component Data Access -> SpawnMarkerEntity Spawn Count Decrement -> Respawn Timer Set

---

## TickingSpawnBeaconSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Managed System (ECS EntityTickingSystem)

### Definition
```java
// Signature
public static class TickingSpawnBeaconSystem extends EntityTickingSystem<EntityStore> {
```

### Architecture & Concepts
This system is a per-tick supervisor that ensures the continued validity of the link between an NPC and its spawn beacon. It runs for every entity that has both a SpawnBeaconReference and an NPCEntity component.

Its core function is to act as a "keep-alive" or "dead man's switch". Each tick, it checks if the reference to the spawn beacon is still valid. If the reference is lost (e.g., the beacon's chunk was unloaded), a timeout counter begins. If this timeout expires and the NPC is not in a "busy" state (like combat), the system flags the NPC for despawning by calling setToDespawn. This is a critical cleanup mechanism that prevents orphaned NPCs from accumulating in the world and consuming server resources.

This system explicitly declares its execution order, running *after* NPCPreTickSystem and *before* DeathSystems.CorpseRemoval, ensuring it operates on up-to-date state information within the tick.

### Lifecycle & Ownership
-   **Creation:** Instantiated and registered with the ECS scheduler during server initialization.
-   **Scope:** Singleton for the duration of the server session.
-   **Destruction:** De-registered and destroyed during server shutdown.

### Internal State & Concurrency
-   **State:** Stateless. The timeout counter it manages is stored within the SpawnBeaconReference component, not the system itself.
-   **Thread Safety:** The system is designed for parallel execution. The isParallel method allows the ECS scheduler to process different chunks of entities on multiple threads simultaneously. The framework guarantees that each call to tick operates on a unique ArchetypeChunk, preventing data races between threads.

### API Surface
The public contract is defined by the EntityTickingSystem base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(1) | Executed once per frame for each matching entity. Manages the spawner-link timeout and triggers despawning if the link is lost. |

### Integration Patterns

#### Standard Usage
This system is not invoked directly. The ECS scheduler automatically calls its tick method for all matching entities every game tick.

#### Anti-Patterns (Do NOT do this)
-   **Incorrect Dependency Ordering:** Modifying the system's dependencies could cause it to act on stale data, for example, attempting to despawn an NPC that has just entered a busy state in the same tick.
-   **External State Modification:** Modifying the timeout counter on the SpawnBeaconReference component from another system can interfere with this system's logic and cause premature or delayed despawning.

### Data Pipeline
> **Flow (Per Tick):**
> Game Loop Tick -> ECS Scheduler -> **TickingSpawnBeaconSystem.tick** -> Check SpawnBeaconReference -> If Invalid, Tick Timeout -> If Timeout Expires, NPCEntity.setToDespawn()

---

## TickingSpawnMarkerSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Managed System (ECS EntityTickingSystem)

### Definition
```java
// Signature
public static class TickingSpawnMarkerSystem extends EntityTickingSystem<EntityStore> {
```

### Architecture & Concepts
This system is functionally identical to the TickingSpawnBeaconSystem but operates on NPCs linked to a SpawnMarkerEntity via the SpawnMarkerReference component. It serves the same critical purpose: to detect and clean up orphaned NPCs whose spawn markers have become invalid or unloaded.

Each tick, it validates the reference to the SpawnMarkerEntity. If the reference is valid, it refreshes timeout counters on both the NPC's reference component and the marker itself. If the reference is lost, it begins a countdown. Upon timeout, and provided the NPC is not in a busy state, it flags the NPC for despawning. This ensures robust cleanup and prevents resource leaks from orphaned entities.

Like its beacon counterpart, it defines a strict execution order to ensure it runs after primary NPC logic and before corpse removal.

### Lifecycle & Ownership
-   **Creation:** Instantiated and registered with the ECS scheduler during server initialization.
-   **Scope:** Singleton for the duration of the server session.
-   **Destruction:** De-registered and destroyed during server shutdown.

### Internal State & Concurrency
-   **State:** Stateless. All state, including timeout counters, is stored in the components of the entities it processes.
-   **Thread Safety:** Designed for parallel execution across ArchetypeChunks, as indicated by the isParallel method. The ECS framework manages thread safety by partitioning the work.

### API Surface
The public contract is defined by the EntityTickingSystem base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(1) | Executed once per frame for each matching entity. Manages the spawner-link timeout and triggers despawning if the link is lost. |

### Integration Patterns

#### Standard Usage
This system is fully automated by the ECS scheduler. Developers do not interact with it directly.

#### Anti-Patterns (Do NOT do this)
-   **Disabling the System:** Removing this system from the server's update loop would lead to a severe bug where NPCs whose spawn markers are unloaded will persist in the world forever, causing performance degradation over time.
-   **State Interference:** Manipulating the timeout counters on the SpawnMarkerReference or SpawnMarkerEntity components from other systems can lead to unpredictable despawning behavior.

### Data Pipeline
> **Flow (Per Tick):**
> Game Loop Tick -> ECS Scheduler -> **TickingSpawnMarkerSystem.tick** -> Check SpawnMarkerReference -> If Invalid, Tick Timeout -> If Timeout Expires, NPCEntity.setToDespawn()


---
description: Architectural reference for SpawnBeaconSystems.ControllerTick
---

# SpawnBeaconSystems.ControllerTick

**Package:** com.hypixel.hytale.server.spawning.beacons
**Type:** System (Transient)

## Definition
```java
// Signature
public static class ControllerTick extends SpawnControllerSystem<NPCBeaconSpawnJob, BeaconSpawnController> {
```

## Architecture & Concepts
The ControllerTick system is the central processing unit for active NPC spawn beacons. It operates within the server's Entity Component System (ECS) framework, executing its logic once per game tick for every entity that represents a live, initialized spawn beacon.

This system acts as the decision-making layer in the spawning pipeline. It does not perform the spawning action itself but is responsible for evaluating the state of the world around a beacon and determining *if*, *when*, and *how many* new NPCs should be spawned.

Its primary responsibilities include:
- **Player Proximity Detection:** It queries the player spatial resource to identify players within the beacon's effective radius. This is the primary trigger for all spawning activity.
- **Spawn Cap Scaling:** It dynamically adjusts the maximum number of concurrent spawns based on the number of nearby players, using a configured ScaledXYResponseCurve. This allows for encounters that scale in difficulty with player density.
- **Spawned NPC Lifecycle Management:** It monitors NPCs previously spawned by the beacon. It will flag NPCs for despawning if they wander too far, remain idle for too long, or if the beacon itself is being deactivated.
- **Beacon Lifecycle Management:** If no players are detected within the beacon's radius for a configurable duration, this system will command the beacon entity to despawn itself to conserve resources.
- **Spawn Job Orchestration:** Based on the above factors, it prepares and queues new spawn jobs for the subsequent SpawnJobSystem to process.

ControllerTick is critically dependent on the output of upstream systems like PlayerSpatialSystem, which must run first to ensure accurate player location data is available.

## Lifecycle & Ownership
- **Creation:** A single instance of ControllerTick is instantiated by the server's core System Registry during engine bootstrap. It is not created on a per-beacon basis.
- **Scope:** The system is a long-lived, stateless object that persists for the entire server session. Its internal state is minimal and reset each tick.
- **Destruction:** The instance is destroyed only when the server shuts down and the ECS framework is dismantled.

## Internal State & Concurrency
- **State:** This system is fundamentally stateless. All operational data is read from components on the beacon entity (such as LegacySpawnBeaconEntity and BeaconSpawnController) or from global resources like WorldTimeResource. It uses a ThreadLocal list of entities for temporary, per-tick processing to avoid heap allocations, but this state does not persist across ticks.
- **Thread Safety:** This system is **not thread-safe**. The implementation explicitly returns false from its isParallel method, signaling to the ECS scheduler that it must be executed serially. This is a deliberate design choice due to its complex, multi-component state modifications and reliance on non-thread-safe collections and resources.

**WARNING:** Attempting to manually schedule this system in a parallel context will lead to race conditions, data corruption, and server instability.

## API Surface
The primary contract is with the ECS scheduler, which invokes the tick method. Direct invocation by user code is not supported.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(N*M) | Executes one cycle of the beacon's logic. N is the number of spawned entities, M is the number of players in range. This is the sole entry point, called by the engine. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. To utilize its functionality, one must create an entity and attach the necessary components to it. The system's query will automatically discover and process the entity on subsequent ticks.

```java
// CORRECT: Create an entity with the required components.
// The ControllerTick system will automatically find and manage it.

Ref<EntityStore> beaconEntity = commandbuffer.createEntity();
commandbuffer.putComponent(beaconEntity, TransformComponent.class, new TransformComponent(position));
commandbuffer.putComponent(beaconEntity, LegacySpawnBeaconEntity.class, new LegacySpawnBeaconEntity("config_id"));

// After the initial setup by LegacyEntityAdded, ControllerTick takes over.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SpawnBeaconSystems.ControllerTick()`. The ECS framework is solely responsible for its lifecycle.
- **Manual Invocation:** Do not call the tick method directly. This bypasses the scheduler, breaks system dependencies, and will cause unpredictable behavior.
- **Component Tampering:** Modifying components like BeaconSpawnController or FloodFillPositionSelector from another thread while this system is running will violate its thread-safety contract and lead to crashes.

## Data Pipeline
This system consumes spatial data and world state to produce commands and spawn jobs.

> Flow:
> PlayerSpatialSystem (updates player locations) -> **ControllerTick** (evaluates beacon, checks players, validates NPCs) -> CommandBuffer (despawns entities) AND BeaconSpawnController (queues spawn jobs) -> SpawnJobTick (executes spawn jobs)

---


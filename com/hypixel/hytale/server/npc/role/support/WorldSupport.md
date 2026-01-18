---
description: Architectural reference for WorldSupport
---

# WorldSupport

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Transient

## Definition
```java
// Signature
public class WorldSupport {
```

## Architecture & Concepts
The WorldSupport class is a stateful, per-NPC component that acts as a facade and performance cache for all interactions between an NPC and the game world. It is not a standalone service; rather, it is an owned, internal mechanism of an NPCEntity, designed to abstract and optimize world queries.

Its primary architectural role is to reduce the cost of frequent environmental queries required for an NPC's decision-making process (e.g., behavior trees, sensors). It achieves this by implementing several caching strategies for world state, such as entity attitudes, block targets, and environmental conditions like weather.

WorldSupport encapsulates the logic for:
- **Perception Caching:** Storing the results of expensive queries like block sensing and entity attitude calculations.
- **State Translation:** Converting raw world data (e.g., chunk environment IDs) into game logic concepts (e.g., current weather).
- **NPC-Specific Context:** All operations are performed from the perspective of the owning NPCEntity. It manages the NPC's relationship with factions, groups, and individual entities.
- **Temporal State:** Manages time-sensitive data, such as temporary attitude overrides which expire after a set duration.

This class is a critical performance component within the server-side NPC system. By centralizing and caching world interactions, it prevents redundant lookups and calculations that would otherwise occur every tick for every active NPC.

### Lifecycle & Ownership
- **Creation:** A WorldSupport instance is created during the initialization of its parent NPCEntity. Specifically, it is instantiated by the NPC's assigned BuilderRole, which passes in the parent entity and a BuilderSupport configuration object. This occurs once when the NPC is first spawned or loaded into the world.
- **Scope:** The lifecycle of a WorldSupport object is strictly bound to its parent NPCEntity. It persists as long as the NPC exists in the world.
- **Destruction:** The object is eligible for garbage collection when its parent NPCEntity is destroyed or unloaded. The `unloaded()` method is invoked as a final cleanup hook to reset internal state, ensuring no stale references are held when the NPC is removed from a chunk.

## Internal State & Concurrency
- **State:** WorldSupport is highly mutable and stateful. Its primary purpose is to maintain a cache of world state relevant to its parent NPC. This includes collections like `attitudeCache`, `blockSensorCachedTargets`, and single-value caches like `cachedEnvironmentId`. The `changeCount` field is used as a simple dirty flag to invalidate caches on a per-tick basis.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be exclusively accessed and manipulated by the main server thread that ticks the parent NPCEntity. All methods assume single-threaded access.
    - **WARNING:** Concurrent modification from other threads will lead to race conditions, inconsistent state, and server instability. Do not share WorldSupport instances across threads or access them from asynchronous tasks without explicit synchronization, which is not provided by the class itself.

## API Surface
The public API provides the interface for an NPC's brain to query the world.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(float dt) | void | O(N) | Advances internal timers. N is the number of active attitude overrides. Must be called every game tick. |
| postRoleBuilt(BuilderSupport) | void | O(N) | Performs second-phase initialization, allocating caches based on NPC configuration. |
| getAttitude(Ref, Ref, ComponentAccessor) | Attitude | O(1) amortized | Retrieves the attitude towards a target entity. Uses a cache to avoid repeated lookups via the AttitudeView. |
| getEnvironmentId(ComponentAccessor) | int | O(1) amortized | Returns the cached environment ID for the NPC's current location. Re-queries the world only once per tick if needed. |
| getCurrentWeatherIndex(ComponentAccessor) | int | O(1) amortized | Returns the cached weather index based on the current environment ID. |
| requestNewPath() | void | O(1) | Sets a flag indicating the NPC's pathfinder should compute a new path. |
| consumeNewPathRequested() | boolean | O(1) | Checks and consumes the new path request flag. This is an atomic read-and-reset operation. |
| overrideAttitude(Ref, Attitude, double) | void | O(1) | Temporarily forces an attitude towards a target for a specified duration. |
| unloaded() | void | O(N) | Resets all internal caches and state. Called when the parent NPC is unloaded. |

## Integration Patterns

### Standard Usage
WorldSupport is an internal dependency of an NPC's role and behavior logic. It is not intended for direct use by external systems. A typical interaction is initiated from a behavior tree node or a sensor.

```java
// Inside an NPC's behavior or sensor logic...
// The 'support' object is the WorldSupport instance owned by the NPC.

// Query the environment to make a decision
ComponentAccessor<EntityStore> accessor = ...;
int environmentId = support.getEnvironmentId(accessor);
if (WeatherRules.isRaining(support.getCurrentWeatherIndex(accessor))) {
    // The NPC decides to seek shelter because it is raining.
    support.requestNewPath();
}

// Determine attitude towards a perceived target
Ref<EntityStore> selfRef = this.parent.getReference();
Ref<EntityStore> targetRef = ...;
Attitude attitude = support.getAttitude(selfRef, targetRef, accessor);
if (attitude.isHostile()) {
    // Initiate combat logic.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldSupport()`. The object is tightly coupled to its parent NPC and requires specific configuration from a BuilderRole during its creation. Incorrect instantiation will result in a non-functional or unstable object.
- **State Sharing:** Do not share a WorldSupport instance between multiple NPCs. Its state is entirely contextual to a single parent entity.
- **Multi-threaded Access:** As stated previously, never call methods on this class from any thread other than the main server tick thread for that NPC. This will corrupt its internal cache state.
- **Ignoring the Tick:** Failing to call the `tick(dt)` method each frame will break time-sensitive features like attitude overrides and periodic cache clearing, potentially leading to memory leaks or stale AI behavior.

## Data Pipeline
WorldSupport primarily acts as a caching layer that sits between the NPC's decision-making logic and the broader world state. It does not process a continuous stream of data but rather serves on-demand, memoized query results.

> **Attitude Query Flow:**
> NPC Behavior Tree -> `WorldSupport.getAttitude(target)` -> **Check `attitudeCache`** -> (Cache Miss) -> `AttitudeView.getAttitude()` -> Blackboard Query -> **`WorldSupport` caches result** -> Return `Attitude`

> **Environment Query Flow:**
> NPC Sensor -> `WorldSupport.getEnvironmentId()` -> **Check `changeCount` validity** -> (Cache Invalid) -> Query `TransformComponent` -> Query `ChunkStore` -> Query `BlockChunk` -> **`WorldSupport` caches Environment ID** -> Return `int`


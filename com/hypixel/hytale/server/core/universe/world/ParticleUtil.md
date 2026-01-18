---
description: Architectural reference for ParticleUtil
---

# ParticleUtil

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Utility

## Definition
```java
// Signature
public class ParticleUtil {
```

## Architecture & Concepts
The ParticleUtil class is a server-side, stateless utility responsible for orchestrating the creation and network distribution of particle effect events. It acts as a high-level, centralized API for any game system that needs to spawn a visual effect, such as combat, block interactions, or environmental effects.

Its core function is to translate a logical request for a particle effect into one or more `SpawnParticleSystem` network packets, which are then dispatched to the relevant clients. It does **not** perform any rendering or simulation itself; it is purely a command dispatcher.

Architecturally, ParticleUtil serves as a bridge between game logic and the networking layer, with deep integration into two key engine systems:
1.  **Entity Component System (ECS):** It uses a ComponentAccessor to retrieve the PacketHandler for each target player entity, which is the final destination for the outgoing network packet.
2.  **Spatial Partitioning System:** For broad, area-of-effect particle spawns, it queries the SpatialResource to efficiently identify all players within a given radius of the effect's origin. This offloads the complex and performance-critical task of proximity detection from the calling game logic.

## Lifecycle & Ownership
- **Creation:** As a static utility class, ParticleUtil is never instantiated. Its bytecode is loaded into the JVM by the class loader upon its first use.
- **Scope:** Application-level. The static methods are available globally throughout the server's runtime.
- **Destruction:** The class is unloaded from memory when the server application shuts down.

## Internal State & Concurrency
- **State:** ParticleUtil is entirely stateless. It contains no mutable fields and all necessary data is provided as method arguments. All operations are idempotent given the same inputs.

- **Thread Safety:** The class itself has no internal state to protect, making its methods inherently safe from race conditions *within this class*.
    - **WARNING:** Thread safety is dictated by the systems it integrates with. Specifically, methods that perform spatial queries (those not explicitly provided a list of players) rely on `SpatialResource.getThreadLocalReferenceList()`. This implies that these methods **must** be called from a thread that has this thread-local storage correctly initialized, which is typically the main world simulation thread. Calling these methods from an asynchronous task or a different worker thread will lead to unpredictable behavior or exceptions.
    - Methods that are explicitly passed a `List<Ref<EntityStore>>` are generally safer to call from other threads, assuming the provided list and the underlying `PacketHandler` are themselves thread-safe.

## API Surface
The API consists of a series of overloaded static methods for spawning particles, differing in the precision of control over position, rotation, and target audience.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| spawnParticleEffect(name, position, componentAccessor) | void | O(log N + k) | Spawns a particle by name at a position. Automatically queries the spatial system to find all players within the default distance (75.0 units). |
| spawnParticleEffect(name, position, playerRefs, componentAccessor) | void | O(k) | Spawns a particle by name at a position, sending it only to a pre-defined list of players. More efficient if the target audience is already known. |
| spawnParticleEffect(particles, position, playerRefs, componentAccessor) | void | O(k) | Spawns a particle defined by a WorldParticle configuration object. This allows for more complex definitions including position and rotation offsets. |
| spawnParticleEffects(particles, position, sourceRef, playerRefs, componentAccessor) | void | O(n*k) | Spawns an array of WorldParticle configurations at a single position for a list of players. |

*N = total number of players in the spatial index, k = number of target players, n = number of particles in the array.*

## Integration Patterns

### Standard Usage
The primary use case is for a server-side system to trigger a visual effect visible to nearby players. The caller must have access to the world's ComponentAccessor.

```java
// Example: A system that creates an effect when a block is broken.
// This code would run on the main server thread.

Vector3d breakPosition = new Vector3d(100.5, 64.0, -50.2);
ComponentAccessor<EntityStore> accessor = world.getComponentAccessor();

// Let the engine find all nearby players automatically.
ParticleUtil.spawnParticleEffect("hytale:block_destroy_stone", breakPosition, accessor);
```

### Anti-Patterns (Do NOT do this)
- **Network Flooding:** Do not call `spawnParticleEffect` in a tight loop or on every single frame for a continuous effect. This will saturate the network connection for clients. Continuous effects should be defined as a single particle system that runs for a duration on the client.
- **Incorrect Threading:** Do not call methods that perform spatial queries from asynchronous tasks or other threads. This will fail due to the dependency on thread-local storage within the spatial system.
- **Redundant Spatial Queries:** If your system logic has already identified a list of target entities (e.g., players in a party or players hit by an AOE spell), do not use the `spawnParticleEffect` overload that performs another spatial query. Pass your existing list of player references directly for maximum performance.

## Data Pipeline
ParticleUtil implements a "fan-out" data flow. It takes a single logical event and broadcasts it as multiple identical packets to a filtered set of clients.

> Flow:
> Game Logic Event -> **ParticleUtil.spawnParticleEffect()** -> (Optional) Spatial Query to find players -> Packet Construction (`SpawnParticleSystem`) -> Iteration over target players -> PlayerRef.getPacketHandler().write() -> Server Network Layer -> Multiple Clients

---


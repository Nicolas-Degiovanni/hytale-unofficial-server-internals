---
description: Architectural reference for BlockPhysicsSystems
---

# BlockPhysicsSystems

**Package:** com.hypixel.hytale.builtin.blockphysics
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class BlockPhysicsSystems {
```

## Architecture & Concepts
The BlockPhysicsSystems class is a static namespace that contains the core components responsible for processing block stability and collapse physics within the game world. It is not instantiated directly but serves as a container for its two critical nested classes: the `Ticking` system and the `CachedAccessor` data access object.

The primary architectural component is **BlockPhysicsSystems.Ticking**, an Entity Component System (ECS) processor that operates on world chunk sections. This system is the engine that drives all non-fluid block physics, such as gravity-affected blocks (sand, gravel) and blocks requiring support (torches, crops).

To achieve high performance, the Ticking system leverages **BlockPhysicsSystems.CachedAccessor**. This is a thread-local cache designed to provide rapid access to the component data of neighboring chunks, a frequent requirement for physics calculations. This pattern avoids costly lookups within the main processing loop.

---

## System: BlockPhysicsSystems.Ticking

This is the central system for processing block physics updates. It is registered with the engine's ECS scheduler and executes once per game tick for each eligible chunk section.

### Definition
```java
// Signature
public static class Ticking extends EntityTickingSystem<ChunkStore> implements DisableProcessingAssert {
```

### Architecture & Concepts
As an `EntityTickingSystem`, this class integrates directly into the server's main tick loop. Its behavior is defined by its query and dependencies.

*   **Query Contract:** The system's `QUERY` targets entities that possess `ChunkSection`, `BlockSection`, `BlockPhysics`, and `FluidSection` components. This ensures it only operates on fully initialized chunk sections capable of supporting physics calculations.
*   **Scheduling Contract:** The `DEPENDENCIES` explicitly order this system to run *after* `ChunkBlockTickSystem.PreTick` and *before* `ChunkBlockTickSystem.Ticking`. This precise placement ensures that blocks scheduled for an update in the current tick are processed before other general-purpose block ticks occur.
*   **Orchestration Role:** This system is an orchestrator, not a direct implementor of physics logic. It contains an optimization to skip sections with no scheduled ticks (`getTickingBlocksCountCopy`). For active sections, it iterates over only the blocks that require an update using `blockSection.forEachTicking`. The actual physics calculations are delegated to the `BlockPhysicsUtil.applyBlockPhysics` method, which receives the pre-fetched, cached neighborhood data from the `CachedAccessor`.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's central ECS Service Manager during world initialization. It is a built-in, non-optional system.
- **Scope:** World-scoped. A single instance persists for the entire lifetime of a game world.
- **Destruction:** The instance is de-referenced and eligible for garbage collection when the world is unloaded.

### Internal State & Concurrency
- **State:** This system is **stateless**. It does not store any data between tick cycles. All necessary state is read directly from entity components during the `tick` method execution.
- **Thread Safety:** The system's stateless nature makes it inherently safe for concurrent execution. The design heavily relies on the `ThreadLocal` `CachedAccessor` to manage state in a thread-safe manner. If the ECS scheduler parallelizes chunk processing across a thread pool, each worker thread will receive its own isolated `CachedAccessor` instance, preventing data corruption and lock contention.

### API Surface
The public API is a contract with the ECS scheduler, not for general developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(N) | Executes one physics tick on a chunk section. N is the number of ticking blocks. Called exclusively by the ECS scheduler. |
| getQuery() | Query | O(1) | Returns the component query that determines which entities this system processes. |
| getDependencies() | Set | O(1) | Returns the set of scheduling dependencies that position this system within the tick loop. |

### Integration Patterns

#### Standard Usage
Developers do not interact with this system directly. Its operation is a side effect of world manipulation. To trigger this system, one must modify a block in a way that schedules its neighbors for a physics update.

```java
// An action like this indirectly causes BlockPhysicsSystems.Ticking to run
// on a subsequent game tick for the affected chunk sections.
world.setBlock(position, Block.AIR);
```

#### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BlockPhysicsSystems.Ticking()`. The system must be managed by the ECS framework to be correctly initialized and scheduled.
- **Manual Invocation:** Do not call the `tick` method directly. This bypasses the ECS scheduler, breaks execution order guarantees, and will lead to unpredictable behavior, race conditions, and world corruption.

### Data Pipeline
The flow of data from a world event to a physics resolution is a multi-stage process orchestrated by several systems.

> Flow:
> World Event (e.g., Player breaks a block) -> `World.setBlock` notifies neighbors -> Neighboring blocks are flagged as "ticking" in their `BlockSection` component -> ECS Scheduler executes **BlockPhysicsSystems.Ticking** on the relevant chunk -> System iterates flagged blocks using a `CachedAccessor` -> `BlockPhysicsUtil.applyBlockPhysics` is called -> Result determines block's fate (e.g., becomes a falling entity, breaks, or stabilizes).

---

## Data Access Pattern: BlockPhysicsSystems.CachedAccessor

This component is a performance-critical, thread-local cache for accessing chunk data from a central point and its neighbors.

### Architecture & Concepts
The `CachedAccessor` solves the "N+1 query problem" for block physics. A single physics check may require reading data from up to 27 different chunk sections (a 3x3x3 volume). Querying the global chunk store for each access would be prohibitively slow.

Instead, the `CachedAccessor` is initialized once per `tick` operation with a given radius. It pre-fetches and caches all required `BlockSection`, `FluidSection`, and `BlockPhysics` components within that radius. Subsequent calls to its `get...` methods are near-instantaneous memory reads from its local cache.

**CRITICAL:** The use of `ThreadLocal` is the cornerstone of its design. It guarantees that each worker thread in a parallelized ticking engine has its own private, reusable cache instance, eliminating the need for locks and preventing race conditions.

### Lifecycle & Ownership
- **Creation:** An instance is created lazily and automatically for each thread the first time it is requested via the `ThreadLocal`.
- **Scope:** Thread-scoped. The instance lives as long as its parent thread.
- **State Invalidation:** The *data* within the accessor is only valid for the duration of a single `tick` operation on a single chunk section. The static `of` factory method must be called at the start of every operation to clear the old state and re-initialize the cache for the new context.

### Anti-Patterns (Do NOT do this)
- **Reference Caching:** Do not store a reference to a `CachedAccessor` in a field or pass it between different top-level operations. Its internal state is transient and will become invalid.
- **Cross-Thread Access:** **NEVER** share a `CachedAccessor` instance between threads. It is explicitly not thread-safe and doing so will cause severe and difficult-to-diagnose data corruption. Always retrieve the thread-specific instance via the static `of` method.


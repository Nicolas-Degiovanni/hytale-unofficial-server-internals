---
description: Architectural reference for LocalSpawnForceTriggerSystem
---

# LocalSpawnForceTriggerSystem

**Package:** com.hypixel.hytale.server.spawning.local
**Type:** System Component

## Definition
```java
// Signature
public class LocalSpawnForceTriggerSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The LocalSpawnForceTriggerSystem is a specialized system within the server-side Entity Component System (ECS) framework responsible for expediting the local spawning cycle for players. It does not perform spawning itself; rather, it acts as a reactive trigger that short-circuits the normal spawn timer in response to a global state change.

Its core architectural function is to decouple the *request* for an immediate spawn check from the *execution* of that check. The system operates on entities possessing both a PlayerRef and a LocalSpawnController component, effectively targeting all players who manage a local spawn volume.

The system's primary tick method is gated by a global resource, LocalSpawnState. It remains dormant until another part of the engine signals a "force trigger" event by modifying this shared resource. Once activated, this system iterates through all relevant player entities and resets their LocalSpawnController's internal timer to a new, short, randomized value. This ensures that the primary LocalSpawningSystem will process these players much sooner than their normal schedule would allow.

**WARNING:** This system is designed for exceptional circumstances, such as a game administrator command or a critical game event requiring an immediate population refresh around a player. It is not part of the standard, time-based spawning loop.

## Lifecycle & Ownership
-   **Creation:** Instantiated once during server or world initialization. It is registered with the main ECS scheduler, which manages its execution. The constructor dependencies (ComponentType, ResourceType) are expected to be resolved by a central registry or dependency injection container.
-   **Scope:** Singleton-per-world. A single instance persists for the entire lifetime of the game world it is registered with.
-   **Destruction:** The instance is discarded and eligible for garbage collection when the corresponding game world is shut down and the ECS scheduler is dismantled.

## Internal State & Concurrency
-   **State:** This class is stateless. Its fields are final references to ECS type definitions, configured at construction and never modified. All stateful operations are performed on components and resources managed externally by the ECS Store.
-   **Thread Safety:** This system is not thread-safe for arbitrary, concurrent invocation. It is designed to be executed exclusively by the single-threaded ECS scheduler. The `tick` methods are guaranteed safe access to the ECS data (Store, ArchetypeChunk) only within the context of the scheduler's execution loop. Any external calls to its methods will lead to state corruption and catastrophic race conditions.

## API Surface
The public API is intended for consumption by the ECS scheduler, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the ECS query for entities with PlayerRef and LocalSpawnController components. |
| tick(dt, systemIndex, store) | void | O(1) | The main system entry point. Checks the global LocalSpawnState resource to decide if per-entity logic should run. |
| tick(dt, index, chunk, store, cmd) | void | O(1) | Per-entity logic. Modifies the LocalSpawnController component to set a new, short time for the next spawn evaluation. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. It is registered with the world's system scheduler and runs automatically. The correct way to trigger its behavior is to acquire the global LocalSpawnState resource and signal a force trigger.

```java
// Somewhere else in the codebase, e.g., a command handler
LocalSpawnState state = world.getStore().getResource(localSpawnStateResourceType);
state.forceTriggerControllers(); // This will cause the system to activate on its next tick
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new LocalSpawnForceTriggerSystem()`. The system must be instantiated and managed by the server's central ECS framework to function correctly.
-   **Manual Invocation:** Do not call any of the `tick` methods directly. This bypasses the ECS scheduler, breaking thread safety guarantees and causing unpredictable behavior.
-   **Assuming Immediacy:** Triggering this system does not cause spawns to happen in the same frame. It merely schedules the next spawn *check* to occur very soon (within 0-5 seconds). Code should not block or wait for spawns to appear after triggering this system.

## Data Pipeline
This system acts as a conditional step in the broader spawning process, activated by a global flag.

> Flow:
> External Event (e.g., Admin Command) -> `LocalSpawnState.forceTriggerControllers()` -> **LocalSpawnForceTriggerSystem** (reads state and modifies component) -> `LocalSpawnController.timeToNextRunSeconds` is updated -> `LocalSpawningSystem` (reads the updated time and executes spawn logic sooner)

---


---
description: Architectural reference for WorldTimeSystems
---

# WorldTimeSystems

**Package:** com.hypixel.hytale.server.core.modules.time
**Type:** Utility / Namespace

## Architecture & Concepts

The WorldTimeSystems class is a static container for two distinct but related systems that manage the lifecycle of world time within the server's Entity Component System (ECS). It does not hold state itself but provides the concrete implementations for initializing and ticking the world's clock.

These inner classes, Init and Ticking, are fundamental components of the server's core game loop. They act as the bridge between the persistent WorldConfig (where game time is saved) and the live, in-memory WorldTimeResource (which is updated every tick). This separation ensures that the high-frequency time simulation is decoupled from the low-frequency persistence layer.

The systems are designed to be registered with an ECS Store, which is typically associated with a specific World instance. They operate exclusively on the WorldTimeResource, a shared resource within that store.

---

# WorldTimeSystems.Init

**Package:** com.hypixel.hytale.server.core.modules.time
**Type:** Transient

## Definition
```java
// Signature
public static class Init extends StoreSystem<EntityStore> {
```

## Architecture & Concepts
The Init system is a lifecycle-aware component responsible for bootstrapping and persisting the world's time. Its sole purpose is to synchronize the in-memory WorldTimeResource with the world's persistent configuration.

-   **On World Load:** When this system is added to a world's ECS Store, the onSystemAddedToStore method is invoked. It reads the last saved game time from the WorldConfig and injects it into the WorldTimeResource, effectively initializing the simulation clock. It also triggers an initial calculation of the moon phase.
-   **On World Unload:** Conversely, when the system is removed from the Store (e.g., during a clean server shutdown or world unload), onSystemRemovedFromStore is called. It extracts the final game time from the WorldTimeResource and writes it back to the WorldConfig, marking it for persistence.

This class guarantees that world time is correctly loaded and saved across server sessions.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's internal module framework during the registration of core game systems. It is not intended for manual creation.
-   **Scope:** The object may be a long-lived singleton within the system registry, but its methods are only executed at specific, infrequent moments: the beginning and end of a world's lifetime in memory.
-   **Destruction:** The instance is garbage collected when the server's system registry is torn down.

## Internal State & Concurrency
-   **State:** This class is stateless. It acts as a transient orchestrator, reading from a WorldConfig and writing to a WorldTimeResource. The worldTimeResourceType field is an immutable reference injected at construction.
-   **Thread Safety:** Not inherently thread-safe. It is designed to be executed by the ECS framework on the main world thread. The call to world.execute demonstrates an awareness of this, ensuring that the moon phase calculation is scheduled on the correct thread. Direct invocation from other threads will lead to race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onSystemAddedToStore(store) | void | O(1) | Loads game time from WorldConfig into the WorldTimeResource. Called by the ECS framework. |
| onSystemRemovedFromStore(store) | void | O(1) | Saves game time from the WorldTimeResource back to the WorldConfig. Called by the ECS framework. |

## Integration Patterns

### Standard Usage
This system is not intended for direct use by developers or module authors. It is automatically registered and managed by the server's core ECS engine when a world is initialized.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of this class using new. The ECS framework is responsible for its creation and dependency injection.
-   **Manual Invocation:** Do not call onSystemAddedToStore or onSystemRemovedFromStore directly. Doing so will bypass the framework's state management and can lead to time desynchronization or data loss.

## Data Pipeline
> **Load Flow:**
> WorldConfig (Persistence) -> **WorldTimeSystems.Init::onSystemAddedToStore** -> WorldTimeResource (In-Memory)
>
> **Save Flow:**
> WorldTimeResource (In-Memory) -> **WorldTimeSystems.Init::onSystemRemovedFromStore** -> WorldConfig (Marked for Persistence)

---

# WorldTimeSystems.Ticking

**Package:** com.hypixel.hytale.server.core.modules.time
**Type:** Transient

## Definition
```java
// Signature
public static class Ticking extends TickingSystem<EntityStore> {
```

## Architecture & Concepts
The Ticking system is the engine of the world clock. As a TickingSystem, it is integrated directly into the server's main game loop and is executed once per tick for every active world.

Its responsibility is extremely narrow: it retrieves the WorldTimeResource from the ECS Store and delegates the tick call to it. The resource itself contains the complex logic for advancing the game time, updating the day-night cycle, and managing other time-dependent phenomena. This class serves as the essential, low-complexity adapter between the generic ECS ticking mechanism and the specific domain logic of time simulation.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's internal module framework alongside other core systems.
-   **Scope:** Its tick method is invoked continuously, once per game tick, for the entire duration that a world is loaded and actively being simulated.
-   **Destruction:** The instance is garbage collected when the server's system registry is torn down.

## Internal State & Concurrency
-   **State:** This class is stateless. Its only field is an immutable reference to the WorldTimeResource type definition.
-   **Thread Safety:** This system is designed to be executed exclusively by the single-threaded ECS scheduler that drives the main game loop. It is not thread-safe and must not be accessed from other threads. All state modifications occur within the WorldTimeResource, which is assumed to be managed by the same single-threaded context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(1) | Delegates the tick operation to the WorldTimeResource. Called every tick by the ECS scheduler. |

## Integration Patterns

### Standard Usage
Like the Init system, this component is fully managed by the engine. Its tick method is called automatically by the server's TickingSystem scheduler. There is no scenario where a developer should interact with this class directly.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never construct an instance of WorldTimeSystems.Ticking.
-   **Manual Ticking:** Calling the tick method manually will advance world time outside of the main game loop, causing severe simulation errors, desynchronization, and undefined behavior.

## Data Pipeline
> **Flow:**
> ECS Scheduler -> **WorldTimeSystems.Ticking::tick** -> WorldTimeResource::tick -> (Internal Time State Updated)


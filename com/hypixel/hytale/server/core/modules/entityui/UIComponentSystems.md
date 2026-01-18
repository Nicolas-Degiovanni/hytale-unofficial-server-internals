---
description: Architectural reference for UIComponentSystems
---

# UIComponentSystems

**Package:** com.hypixel.hytale.server.core.modules.entityui
**Type:** Utility / System Container

## Definition
```java
// Signature
public class UIComponentSystems {
    // Contains nested static System classes: Setup, Update, Remove
}
```

## Architecture & Concepts
UIComponentSystems is a static container class that groups three distinct but related Entity Component Systems (ECS) responsible for managing the server-side lifecycle and network synchronization of entity-attached UI components. These systems ensure that clients receive the correct UI information for entities they can see, and that this information is correctly initialized and cleaned up.

This class acts as the authoritative backend for what players see on their screen for in-world UI elements attached to entities. It directly integrates with the Entity Tracker to make decisions based on player visibility.

The three core systems are:
- **Setup:** An initialization system that ensures any relevant entity is given a UIComponentList upon its creation. This guarantees a consistent state for all entities that are capable of displaying UI.
- **Update:** A ticking system that detects when an entity becomes newly visible to a player. It then serializes the entity's UI component data and queues it for network transmission to that specific player.
- **Remove:** A change-detection system that triggers when a UIComponentList is removed from an entity. It sends a corresponding removal command to all players who could previously see the UI, ensuring it is properly destroyed on the client.

## Lifecycle & Ownership
The UIComponentSystems class itself is a static container and is not instantiated. Its lifecycle is tied to the JVM. The inner system classes, however, have a managed lifecycle within the server's ECS framework.

- **Creation:** Instances of UIComponentSystems.Setup, UIComponentSystems.Update, and UIComponentSystems.Remove are created once during server bootstrap. They are then registered with the central ECS System Scheduler.
- **Scope:** These system instances are singletons within the ECS world and persist for the entire duration of the server session.
- **Destruction:** The systems are discarded only when the server shuts down and the ECS world is destroyed.

## Internal State & Concurrency
The top-level UIComponentSystems class is stateless. The nested system classes are stateful but their state is configured at creation and is immutable thereafter.

- **State:** Each inner system holds immutable references to ComponentType objects, which act as handles or keys for querying component data. They do not maintain any mutable state across ticks or events. All state changes are performed on components within the ECS Store via CommandBuffers.
- **Thread Safety:**
    - **Update:** This system is explicitly designed for parallel execution. The `isParallel` method allows the ECS scheduler to process different ArchetypeChunks on separate threads, significantly improving performance on multi-core processors. All operations are performed on chunk-local data, avoiding cross-thread contamination.
    - **Setup & Remove:** These are event-driven systems (onEntityAdd, onComponentRemoved). The ECS framework guarantees that these hooks are executed in a thread-safe context, typically serially within a specific phase of the game loop. They are not designed to be invoked concurrently by user code.

---

## System: UIComponentSystems.Setup
This system ensures that all living entities are properly initialized with a UIComponentList.

### Definition
```java
// Signature
public static class Setup extends HolderSystem<EntityStore> {
```

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback executed by the ECS framework when an entity matching the query is added. Ensures a UIComponentList exists. |
| getQuery() | Query | O(1) | Returns a query for all legacy living entities, defining the scope of this system. |

---

## System: UIComponentSystems.Update
This system is responsible for synchronizing UI state to clients when an entity becomes visible.

### Definition
```java
// Signature
public static class Update extends EntityTickingSystem<EntityStore> {
```

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(N) | Executed each tick for every entity matching the query. Checks for newly visible players and queues a ComponentUpdate packet. N is the number of newly visible players. |
| getGroup() | SystemGroup | O(1) | Assigns this system to the QUEUE_UPDATE_GROUP, ensuring it runs after visibility calculations are complete but before network packets are sent. |
| getQuery() | Query | O(1) | Returns a query for entities that are both visible and have a UIComponentList. |

---

## System: UIComponentSystems.Remove
This system ensures that a client is instructed to destroy an entity's UI when the server removes the corresponding component.

### Definition
```java
// Signature
public static class Remove extends RefChangeSystem<EntityStore, UIComponentList> {
```

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentRemoved(ref, comp, store, cmd) | void | O(N) | Callback executed when a UIComponentList is removed from an entity. Queues a removal command for all players (N) who can see the entity. |
| getQuery() | Query | O(1) | Returns a query for entities that are both visible and have a UIComponentList. |

---

## Integration Patterns

### Standard Usage
These systems are not intended to be invoked directly. They must be registered with the server's primary ECS scheduler during the server initialization phase. The framework is then responsible for invoking their lifecycle methods at the correct time.

```java
// Conceptual example of system registration during server startup
SystemScheduler scheduler = server.getECSScheduler();

// Component types are retrieved from a central registry
ComponentType<EntityStore, EntityTrackerSystems.Visible> visibleType = ...;
ComponentType<EntityStore, UIComponentList> uiListType = ...;

// Systems are instantiated and added to the scheduler
scheduler.addSystem(new UIComponentSystems.Setup(uiListType));
scheduler.addSystem(new UIComponentSystems.Update(visibleType, uiListType));
scheduler.addSystem(new UIComponentSystems.Remove(visibleType, uiListType));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Never create an instance of these systems and call methods like `tick` or `onEntityAdd` manually. This bypasses the ECS scheduler, breaking thread safety guarantees and causing unpredictable behavior.
- **Incorrect Registration Order:** The `Update` system depends on the `EntityTrackerSystems` to have run first. Registering it outside of the `QUEUE_UPDATE_GROUP` can lead to race conditions where UI updates are sent based on stale visibility data.

## Data Pipeline
The primary data flow concerns the `Update` system, which propagates server state to clients.

> Flow:
> 1. **Entity Tracker System** runs, calculating which players can see which entities. It populates the `newlyVisibleTo` map inside the `EntityTrackerSystems.Visible` component.
> 2. **UIComponentSystems.Update** runs. Its `tick` method iterates over entities with a `Visible` component.
> 3. If `newlyVisibleTo` is not empty, the system creates a `ComponentUpdate` network packet.
> 4. The packet is populated with the entity's UI component IDs from its `UIComponentList`.
> 5. This packet is queued on the `EntityViewer` object for each new viewer.
> 6. The server's **Network System** later collects these queued packets from all `EntityViewer`s and sends them to the appropriate clients.


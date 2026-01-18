---
description: Architectural reference for NameplateSystems
---

# NameplateSystems

**Package:** com.hypixel.hytale.server.core.entity.nameplate
**Type:** Utility

## Definition
```java
// Signature
public class NameplateSystems {
    // Contains nested static system classes
}
```

## Architecture & Concepts
NameplateSystems is a static container class that provides a namespace for Entity Component Systems (ECS) related to the network synchronization of the Nameplate component. It does not hold state or contain logic itself; rather, it groups the systems responsible for managing the lifecycle of nameplate data as it is replicated to clients.

These systems form a critical link between the server-side game state and the client-side presentation layer. They are designed to operate within the server's main tick loop, reacting to changes in entity components and ensuring that players only receive updates for entities they are able to see, as determined by the EntityTrackerSystems.

The primary responsibilities of the contained systems are:
- **Update Propagation:** Detecting changes to a Nameplate component and creating network packets to inform relevant clients.
- **Visibility Management:** Sending initial nameplate data to clients when an entity becomes visible to them.
- **Cleanup:** Notifying clients to destroy a nameplate renderable when the corresponding entity is no longer visible or the Nameplate component is removed.

---

# NameplateSystems.EntityTrackerRemove

**Package:** com.hypixel.hytale.server.core.entity.nameplate
**Type:** ECS System

## Definition
```java
// Signature
public static class EntityTrackerRemove extends RefChangeSystem<EntityStore, Nameplate> {
```

## Architecture & Concepts
EntityTrackerRemove is a reactive ECS system that triggers when a Nameplate component is removed from an entity. It extends RefChangeSystem, a specialized system type that listens for component addition, modification, or removal events rather than running on every tick.

Its sole responsibility is to handle the cleanup of nameplate data on the client. When a server-side Nameplate component is removed, this system queries the entity's visibility information via the EntityTrackerSystems.Visible component. It then iterates through every client (viewer) that can currently see the entity and queues a specific "remove" command for the nameplate. This ensures that clients do not retain a stale nameplate for an entity that should no longer have one.

This system is essential for maintaining state consistency between the server and connected clients.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's central ECS registry during the world initialization phase. System definitions are typically discovered automatically.
- **Scope:** The system's lifecycle is tied to the EntityStore it operates on. It persists as long as the server world is active.
- **Destruction:** Destroyed and garbage collected when the server world is shut down and the associated EntityStore is de-allocated.

## Internal State & Concurrency
- **State:** The system is stateless. Its fields, such as componentType and query, are immutable references configured at construction time. It does not cache or store any per-entity data.
- **Thread Safety:** This system is not designed for concurrent execution. RefChangeSystem callbacks are typically invoked from a single thread during a specific phase of the game loop after all other systems have finished their updates and command buffers are being processed. Direct concurrent invocation is an anti-pattern and will lead to race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentRemoved(...) | void | O(N) | **Framework-Internal.** Callback invoked by the ECS when a Nameplate is removed. N is the number of viewers. Queues a network packet for each viewer. |
| getQuery() | Query | O(1) | **Framework-Internal.** Returns the component query that determines which entities this system operates on. |
| componentType() | ComponentType | O(1) | **Framework-Internal.** Specifies the component type this RefChangeSystem is observing. |

## Integration Patterns

### Standard Usage
This system is not used directly. It is automatically engaged by the ECS framework. A developer triggers its logic by removing a Nameplate component from an entity.

```java
// A different system or game logic code would execute this.
// EntityTrackerRemove will be automatically invoked by the framework.
commandBuffer.removeComponent(entityRef, Nameplate.class);
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call onComponentRemoved directly. This bypasses the ECS framework's state management and will cause unpredictable behavior.
- **Direct Instantiation:** Do not use new EntityTrackerRemove(). Systems must be managed by the server's central system registry to be correctly integrated into the game loop.

## Data Pipeline
> Flow:
> Game Logic -> CommandBuffer.removeComponent -> ECS Executor -> **EntityTrackerRemove.onComponentRemoved** -> EntityViewer.queueRemove -> Client Network Buffer

---

# NameplateSystems.EntityTrackerUpdate

**Package:** com.hypixel.hytale.server.core.entity.nameplate
**Type:** ECS System

## Definition
```java
// Signature
public static class EntityTrackerUpdate extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
EntityTrackerUpdate is a proactive ECS system that runs once per tick for every entity that has both a Nameplate and an EntityTrackerSystems.Visible component. It extends EntityTickingSystem, the standard base class for per-entity update logic.

The core function of this system is to synchronize the state of the Nameplate component with all relevant clients. It performs two key checks during its tick method:
1. **Dirty Check:** It inspects the Nameplate component for a "network outdated" flag. If this flag is set (e.g., by `nameplate.setText()`), the system serializes the nameplate data and broadcasts it to all current viewers.
2. **New Viewer Check:** It checks if any new clients have entered the entity's visibility set during the current tick. If so, it sends the full nameplate data to these new viewers, ensuring they receive the correct initial state.

This system is designed for high performance and can operate in parallel across multiple threads, as indicated by the isParallel method.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's central ECS registry during world initialization.
- **Scope:** The system's lifecycle is tied to the EntityStore it operates on. It persists as long as the server world is active.
- **Destruction:** Destroyed when the server world is shut down.

## Internal State & Concurrency
- **State:** This system is stateless. It holds immutable references to component types and its query. All state is read directly from the components of the entity being processed.
- **Thread Safety:** The tick method is thread-safe and designed for parallel execution. It operates on a distinct chunk of entities and writes its output to thread-safe command queues (via EntityViewer). The `consumeNetworkOutdated` method on the Nameplate component is an atomic operation that prevents multiple updates for the same change.

**WARNING:** The thread safety of this system relies on the Nameplate component's internal state being managed atomically (e.g., via AtomicBoolean) and the EntityViewer's queueing methods being thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(N) | **Framework-Internal.** Executes once per entity per frame. N is the number of viewers. Checks for state changes and queues network updates. |
| getQuery() | Query | O(1) | **Framework-Internal.** Returns the component query that selects entities with both Nameplate and Visible components. |
| getGroup() | SystemGroup | O(1) | **Framework-Internal.** Assigns this system to a specific execution group (QUEUE_UPDATE_GROUP), ensuring it runs after visibility calculations are complete. |

## Integration Patterns

### Standard Usage
This system is not used directly. A developer triggers its logic by modifying the state of a Nameplate component on an entity.

```java
// In another system, get the Nameplate component and modify it.
// EntityTrackerUpdate will automatically detect and send the change.
Nameplate nameplate = store.getComponent(entityRef, Nameplate.class);
if (nameplate != null) {
    nameplate.setText("New Name"); // This sets an internal dirty flag
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Tick:** Never call the tick method manually. The ECS scheduler is responsible for invoking it with the correct context and data.
- **State Modification:** Do not attempt to modify the Nameplate component from a different thread without proper synchronization. While this system is thread-safe, the component itself may not be. All component mutations should be done via CommandBuffers from the main thread.

## Data Pipeline
> Flow:
> Game Logic -> Nameplate.setText() -> `networkOutdated` flag set -> **EntityTrackerUpdate.tick()** -> ComponentUpdate created -> EntityViewer.queueUpdate -> Client Network Buffer


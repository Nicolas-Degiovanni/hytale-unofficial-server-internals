---
description: Architectural reference for EntitySystems
---

# EntitySystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility / Namespace

## Architecture & Concepts
The EntitySystems class is not a traditional object but rather a static namespace that groups together several related, fine-grained systems within the Hytale Entity Component System (ECS) framework. Each nested class represents a distinct piece of logic that operates on entities possessing a specific combination of components.

These systems are fundamental to entity lifecycle management, state synchronization, and cleanup. They handle tasks ranging from transient marker component removal to network state propagation for clients. The primary architectural pattern is the separation of data (Components) from logic (Systems), allowing for highly parallelizable and maintainable game logic.

---

## ClearMarker (Abstract)

**Type:** System

### Definition
```java
// Signature
public abstract static class ClearMarker<T extends Component<EntityStore>> extends RefSystem<EntityStore> {
```

### Architecture & Concepts
ClearMarker is an abstract base system designed to immediately remove a specific "marker" component from an entity as soon as it is added to the world. This pattern is used to tag entities for special processing by other systems during the same tick they are created, without the tag persisting and causing side effects in subsequent ticks.

For example, an entity spawned from a prefab might be given a FromPrefab component. Systems can query for this component to perform prefab-specific initialization. The ClearMarker system then runs, removing the FromPrefab component, ensuring the initialization logic runs only once.

The system's execution order is explicitly managed via dependencies to run *after* a designated SystemGroup, ensuring that other systems have a chance to react to the marker component before it is removed.

### Lifecycle & Ownership
- **Creation:** Instantiated by the ECS framework's system registrar during server bootstrap. Concrete implementations like ClearFromPrefabMarker are registered.
- **Scope:** Session-scoped. The system instance persists for the entire server lifetime.
- **Destruction:** Decommissioned during server shutdown.

### Internal State & Concurrency
- **State:** Stateless. The system holds no mutable state between executions. Its behavior is determined entirely by the components of the entity it processes.
- **Thread Safety:** Fully thread-safe. The system operates on entities via a CommandBuffer, which safely queues component removal operations for later execution by the ECS framework.

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(ref, reason, store, commandBuffer) | void | O(1) | Callback triggered when an entity matching the query is added. Immediately queues the removal of the marker component. |
| getQuery() | Query | O(1) | Returns the query that selects entities possessing the specific marker component. |
| getDependencies() | Set | O(1) | Defines the execution order, ensuring this system runs after other systems have processed the marker. |

### Integration Patterns

#### Standard Usage
This system is not used directly. It is registered with the ECS world, which automatically invokes it on matching entities. The pattern is to add a marker component to an entity upon creation.

```java
// An entity is created with a marker component
commandBuffer.addComponent(newEntity, new FromPrefab());

// Later in the tick, after other systems have run,
// the ClearFromPrefabMarker system will automatically
// execute and remove the FromPrefab component.
```

---

## DynamicLightTracker

**Type:** System

### Definition
```java
// Signature
public static class DynamicLightTracker extends EntityTickingSystem<EntityStore> {
```

### Architecture & Concepts
DynamicLightTracker is a ticking system responsible for synchronizing dynamic lighting information from server-side entities to relevant clients. It queries for entities that are both visible to players (possess an EntityTrackerSystems.Visible component) and emit light (possess a DynamicLight component).

The system's primary function is to detect changes in an entity's lighting state or visibility. When a change is detected (e.g., light color changes, or a player moves into range of the light-emitting entity), it queues a network update packet for all affected clients. This ensures that players see a consistent and updated view of the world's lighting.

It operates within the EntityTrackerSystems.QUEUE_UPDATE_GROUP, positioning its execution correctly within the broader entity tracking and network synchronization pipeline.

### Lifecycle & Ownership
- **Creation:** Instantiated and registered by the ECS framework during server bootstrap.
- **Scope:** Session-scoped. Persists for the entire server lifetime.
- **Destruction:** Decommissioned during server shutdown.

### Internal State & Concurrency
- **State:** Stateless. All necessary information is read directly from entity components during the tick.
- **Thread Safety:** Parallelizable. The system's tick method can be executed concurrently across multiple threads, as indicated by the isParallel method. The ECS framework guarantees that each thread operates on a distinct set of entities (ArchetypeChunks), preventing race conditions.

### Data Pipeline
> Flow:
> DynamicLight Component State Change -> **DynamicLightTracker** (detects change) -> EntityViewer.queueUpdate() -> Network Packet Queue -> Client Render Update

---

## NewSpawnTick

**Type:** System

### Definition
```java
// Signature
public static class NewSpawnTick extends EntityTickingSystem<EntityStore> {
```

### Architecture & Concepts
NewSpawnTick is a simple but critical housekeeping system. Its sole responsibility is to remove the NewSpawnComponent from entities after a short time window has elapsed since their creation.

The NewSpawnComponent is a marker used to signal that an entity has just been introduced into the world. Other systems, like NewSpawnEntityTrackerUpdate, use this component to trigger special, one-time logic, such as sending a specific network packet to clients. NewSpawnTick acts as the garbage collector for this temporary component, ensuring it does not persist indefinitely.

### Lifecycle & Ownership
- **Creation:** Instantiated and registered by the ECS framework during server bootstrap.
- **Scope:** Session-scoped. Persists for the entire server lifetime.
- **Destruction:** Decommissioned during server shutdown.

### Internal State & Concurrency
- **State:** Stateless.
- **Thread Safety:** Parallelizable. The system's tick method is designed for concurrent execution across different entity chunks. Component removal is safely handled via the CommandBuffer.

### Integration Patterns

#### Standard Usage
This system works automatically in the background. A developer's only interaction with this logic is to add a NewSpawnComponent to an entity upon its creation.

```java
// When spawning a new entity:
commandBuffer.addComponent(newEntity, new NewSpawnComponent());

// The NewSpawnTick system will automatically remove this component
// after its internal timer expires a few ticks later.
```

---

## OnLoadFromExternal

**Type:** System

### Definition
```java
// Signature
public static class OnLoadFromExternal extends HolderSystem<EntityStore> {
```

### Architecture & Concepts
OnLoadFromExternal is an event-driven system that assigns a stable, unique identifier (UUID) to entities that are being loaded from an external source, such as a prefab or world generation. It queries for entities that have either a FromPrefab or FromWorldGen marker component.

Upon detection of such an entity being added to the world, this system generates a new version 3 UUID and attaches it via a UUIDComponent. This is a critical step for ensuring that entities loaded from disk or created by procedural generation can be uniquely and persistently identified across sessions and network clients.

The system's dependencies ensure it runs *before* the standard EntityStore.UUIDSystem but *after* any legacy UUID systems, placing it correctly in the entity initialization pipeline.

### Lifecycle & Ownership
- **Creation:** Instantiated and registered by the ECS framework during server bootstrap.
- **Scope:** Session-scoped. Persists for the entire server lifetime.
- **Destruction:** Decommissioned during server shutdown.

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback triggered when a matching entity is added. Generates and attaches a UUIDComponent. |

### Anti-Patterns (Do NOT do this)
- **Manual UUID Assignment:** Do not manually add a UUIDComponent to entities that are also marked with FromPrefab or FromWorldGen. This can lead to identity conflicts or race conditions with this system. Allow this system to be the sole authority for UUID assignment in these cases.

---

## UnloadEntityFromChunk

**Type:** System

### Definition
```java
// Signature
public static class UnloadEntityFromChunk extends RefSystem<EntityStore> {
```

### Architecture & Concepts
UnloadEntityFromChunk is a cleanup system that manages the relationship between an entity and the world chunk it resides in. It listens for the removal of any entity that has a TransformComponent.

When an entity is removed or unloaded, this system is triggered. It retrieves the entity's last known chunk reference from its TransformComponent and then updates the corresponding EntityChunk component on that chunk entity. This involves removing the entity's reference from the chunk's list of contained entities.

This process is vital for maintaining data integrity. Without it, chunks would retain stale references to destroyed entities, leading to memory leaks, corrupted game state, and potential crashes when iterating over a chunk's entity list.

### Lifecycle & Ownership
- **Creation:** Instantiated and registered by the ECS framework during server bootstrap.
- **Scope:** Session-scoped. Persists for the entire server lifetime.
- **Destruction:** Decommissioned during server shutdown.

### Data Pipeline
> Flow:
> Entity Unload/Remove Event -> **UnloadEntityFromChunk** (triggered) -> Retrieves TransformComponent -> Retrieves Chunk Reference -> Updates EntityChunk Component on Chunk Entity


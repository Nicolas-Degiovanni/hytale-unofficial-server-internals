---
description: Architectural reference for PrefabPathSystems
---

# PrefabPathSystems

**Package:** com.hypixel.hytale.builtin.path
**Type:** Utility

## Definition
```java
// Signature
public class PrefabPathSystems {
```

## Architecture & Concepts
The PrefabPathSystems class is not a traditional object but a static container, acting as a namespace for a suite of related Entity-Component-System (ECS) classes. These inner systems collectively manage the entire lifecycle and behavior of patrol path entities within the game world.

The primary responsibility of this collection is to bridge the gap between raw entity data (the PatrolPathMarkerEntity component) and the logical, world-level path structures managed by the WorldPathData resource. These systems handle entity creation, destruction, data migration, nameplate display, and the critical remapping of path identifiers when prefabs are placed in the world.

Each inner class is a distinct system registered with the game engine's main loop, designed to react to specific events or state changes in a data-driven manner.

---
# PrefabPathSystems.AddOrRemove

**Package:** com.hypixel.hytale.builtin.path
**Type:** Transient

## Definition
```java
// Signature
public static class AddOrRemove extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This system is the primary lifecycle manager for path marker entities. It operates on any entity that possesses a PatrolPathMarkerEntity component. Its core function is to register a newly created path marker entity with its corresponding logical path object, which is stored in the global WorldPathData resource. Conversely, it deregisters the marker when the entity is removed.

This class is critical for maintaining the integrity of the path data structures. It also performs essential setup tasks on entity creation, such as assigning a visible model and ensuring the entity is correctly configured for in-game tools.

A key feature is its built-in data migration logic. It detects legacy path markers that lack a UUID-based PathId and generates one, ensuring backward compatibility with older world data. The system is explicitly ordered to run *before* ModelSystems, guaranteeing that the path marker's model is attached before any rendering logic attempts to process it.

### Lifecycle & Ownership
- **Creation:** Instantiated by the engine's SystemRegistry during server world initialization. One instance exists per world.
- **Scope:** Persists for the entire duration of a world session.
- **Destruction:** The instance is discarded when the server world is shut down or unloaded.

## Internal State & Concurrency
- **State:** This system is entirely stateless. All operations are performed on the components of the entity being processed or on the WorldPathData resource, which are passed as arguments to its methods.
- **Thread Safety:** Not thread-safe. The Hytale ECS framework guarantees that all system methods are executed serially on the main world thread. Invoking its methods from other threads will lead to race conditions and world corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(log N) | Invoked when an entity with PatrolPathMarkerEntity is created. Registers the waypoint with WorldPathData, assigns a model, and performs data migration if needed. |
| onEntityRemoved(holder, reason, store) | void | O(log N) | Invoked when a path marker entity is destroyed. Removes or unloads the waypoint from its logical path in WorldPathData based on the reason for removal. |

## Integration Patterns

### Standard Usage
This system is automatically registered and invoked by the ECS engine. Developers do not interact with it directly. Its logic is triggered implicitly when a path marker entity is spawned or despawned.

```java
// Engine-level pseudo-code
// Spawning an entity with this component triggers the system
Entity entity = world.createEntity();
entity.addComponent(new PatrolPathMarkerEntity(...));
// AddOrRemove.onEntityAdd is now automatically invoked by the engine
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AddOrRemove()`. The ECS framework manages its lifecycle.
- **Manual Invocation:** Do not call `onEntityAdd` or `onEntityRemoved` directly. Doing so bypasses the engine's state management and will cause data desynchronization.

## Data Pipeline
> Flow (Creation):
> Entity with PatrolPathMarkerEntity is spawned -> ECS Query Match -> **AddOrRemove.onEntityAdd** -> Accesses WorldPathData Resource -> Registers Waypoint -> Adds ModelComponent, HiddenFromAdventurePlayers, PrefabCopyableComponent to Entity

> Flow (Deletion):
> Entity is destroyed -> ECS Notification -> **AddOrRemove.onEntityRemoved** -> Accesses WorldPathData Resource -> Unregisters Waypoint

---
# PrefabPathSystems.AddedFromWorldGen

**Package:** com.hypixel.hytale.builtin.path
**Type:** Transient

## Definition
```java
// Signature
public static class AddedFromWorldGen extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This is a specialized, preparatory system designed to handle path markers spawned directly by the world generation process. World generation entities are often tagged with a temporary FromWorldGen component that contains a world generation identifier.

The sole responsibility of this system is to identify such entities, extract the identifier, and promote it into a permanent WorldGenId component. Crucially, its dependency configuration ensures it executes *before* the main AddOrRemove system. This guarantees that when AddOrRemove runs, the permanent WorldGenId is already present, allowing for the correct generation of path UUIDs that are unique to their world generation context.

### Lifecycle & Ownership
- **Creation:** Instantiated by the engine's SystemRegistry during server world initialization.
- **Scope:** Persists for the entire duration of a world session.
- **Destruction:** Discarded when the server world is unloaded.

## Internal State & Concurrency
- **State:** Stateless.
- **Thread Safety:** Not thread-safe. Must only be executed by the main world thread via the ECS scheduler.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Invoked when an entity with both PatrolPathMarkerEntity and FromWorldGen is created. Copies the ID from FromWorldGen to a new WorldGenId component. |

## Data Pipeline
> Flow:
> World Generator spawns Entity with FromWorldGen -> ECS Query Match -> **AddedFromWorldGen.onEntityAdd** -> Adds WorldGenId component -> AddOrRemove system runs next

---
# PrefabPathSystems.NameplateHolderSystem

**Package:** com.hypixel.hytale.builtin.path
**Type:** Transient

## Definition
```java
// Signature
public static class NameplateHolderSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This system ensures that every path marker entity has a visible nameplate for use in creative or editing modes. When a path marker is created, this system checks for the existence of a DisplayNameComponent and a Nameplate component.

If a display name is missing, it attempts to restore a legacy name or creates a default one. It then ensures a Nameplate component exists, populating it with the display name text. This decouples the entity's logical name from the component responsible for rendering it.

### Lifecycle & Ownership
- **Creation:** Instantiated by the engine's SystemRegistry during server world initialization.
- **Scope:** Persists for the entire duration of a world session.
- **Destruction:** Discarded when the server world is unloaded.

## Internal State & Concurrency
- **State:** Stateless.
- **Thread Safety:** Not thread-safe. Must only be executed by the main world thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | On entity creation, ensures DisplayNameComponent and Nameplate components exist, creating them with default or legacy values if necessary. |

---
# PrefabPathSystems.NameplateRefChangeSystem

**Package:** com.hypixel.hytale.builtin.path
**Type:** Transient

## Definition
```java
// Signature
public static class NameplateRefChangeSystem extends RefChangeSystem<EntityStore, DisplayNameComponent> {
```

## Architecture & Concepts
This is a highly efficient, reactive system that synchronizes the text of a path marker's nameplate with its DisplayNameComponent. Unlike a HolderSystem that runs only on creation, a RefChangeSystem is triggered by fine-grained changes to a specific component type.

Whenever the DisplayNameComponent on a path marker entity is added, removed, or modified, this system's corresponding method is invoked. It reads the new display name and updates the text field of the Nameplate component. This is a classic observer pattern implementation within the ECS, ensuring that the UI representation (Nameplate) is always consistent with the underlying data (DisplayNameComponent) without wasteful polling.

### Lifecycle & Ownership
- **Creation:** Instantiated by the engine's SystemRegistry during server world initialization.
- **Scope:** Persists for the entire duration of a world session.
- **Destruction:** Discarded when the server world is unloaded.

## Internal State & Concurrency
- **State:** Stateless.
- **Thread Safety:** Not thread-safe. The ECS framework invokes its methods on the main world thread in response to component changes.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(...) | void | O(1) | When DisplayNameComponent is added, updates the Nameplate text. |
| onComponentSet(...) | void | O(1) | When DisplayNameComponent is replaced, updates the Nameplate text. |
| onComponentRemoved(...) | void | O(1) | When DisplayNameComponent is removed, clears the Nameplate text. |

---
# PrefabPathSystems.PrefabPlaceEntityEventSystem

**Package:** com.hypixel.hytale.builtin.path
**Type:** Transient

## Definition
```java
// Signature
public static class PrefabPlaceEntityEventSystem extends WorldEventSystem<EntityStore, PrefabPlaceEntityEvent> {
```

## Architecture & Concepts
This system is critical for making paths function correctly within reusable prefabs. It listens for a global PrefabPlaceEntityEvent, which is fired by the engine whenever an entity is created as part of a prefab being pasted into the world.

When this event occurs for a path marker entity, the system's primary job is to remap its path identifier. It takes the original PathId stored in the prefab and requests a new, unique ID from the BuilderToolsPlugin. This prevents UUID collisions between paths from two separate instances of the same prefab. This remapping ensures that each pasted prefab contains a new, distinct path, rather than incorrectly extending the path of a previously placed prefab.

### Lifecycle & Ownership
- **Creation:** Instantiated by the engine's SystemRegistry during server world initialization.
- **Scope:** Persists for the entire duration of a world session.
- **Destruction:** Discarded when the server world is unloaded.

## Internal State & Concurrency
- **State:** Stateless.
- **Thread Safety:** Not thread-safe. Event handlers are invoked serially on the main world thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(store, commandBuffer, event) | void | O(1) | Handles the PrefabPlaceEntityEvent. If the entity is a path marker, it generates a new unique PathId for it. |

## Data Pipeline
> Flow:
> User pastes a prefab -> Engine fires PrefabPlaceEntityEvent -> **PrefabPlaceEntityEventSystem.handle** -> Calls BuilderToolsPlugin to get new UUID -> Updates PathId on the entity's PatrolPathMarkerEntity component

---
# PrefabPathSystems.WorldGenChangeSystem

**Package:** com.hypixel.hytale.builtin.path
**Type:** Transient

## Definition
```java
// Signature
public static class WorldGenChangeSystem extends RefChangeSystem<EntityStore, WorldGenId> {
```

## Architecture & Concepts
This system is analogous to the NameplateRefChangeSystem but reacts to changes in the WorldGenId component instead. Its purpose is to keep the entity's display name synchronized with its world generation context.

Whenever a WorldGenId component is added or changed on a path marker entity, this system regenerates the entity's display name using a static helper method. This ensures the nameplate text, for example "wg-123-MyPath-1", accurately reflects the entity's origin. This is essential for developers and designers to debug and identify paths that were spawned by specific world generation rules.

### Lifecycle & Ownership
- **Creation:** Instantiated by the engine's SystemRegistry during server world initialization.
- **Scope:** Persists for the entire duration of a world session.
- **Destruction:** Discarded when the server world is unloaded.

## Internal State & Concurrency
- **State:** Stateless.
- **Thread Safety:** Not thread-safe. The ECS framework invokes its methods on the main world thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(...) | void | O(1) | When WorldGenId is added, regenerates and sets the DisplayNameComponent. |
| onComponentSet(...) | void | O(1) | When WorldGenId is changed, regenerates and sets the DisplayNameComponent. |
| onComponentRemoved(...) | void | O(1) | When WorldGenId is removed, sets the DisplayNameComponent to a default "unknown" value. |


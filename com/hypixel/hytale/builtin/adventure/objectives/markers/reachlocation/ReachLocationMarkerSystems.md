---
description: Architectural reference for ReachLocationMarkerSystems
---

# ReachLocationMarkerSystems

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers.reachlocation
**Type:** Utility / System Container

## Definition
```java
// Signature
public class ReachLocationMarkerSystems {
    // Contains nested static System classes
}
```

## Architecture & Concepts

The ReachLocationMarkerSystems class is a container for a group of related ECS Systems that collectively manage the lifecycle and logic for "reach location" objective markers. This class does not have its own state or behavior; it acts as a namespace and organizational unit for three distinct, highly-specialized systems that operate on entities possessing a ReachLocationMarker component.

These systems form the bridge between the abstract objective framework and the concrete in-world representation of a location marker. They are responsible for detecting when a player physically enters a marker's trigger radius and propagating that event to the objective system to advance quest state.

The three core systems are:

*   **EnsureNetworkSendable:** A housekeeping system that guarantees any entity with a ReachLocationMarker component is also assigned a NetworkId. This is a critical prerequisite for the marker entity to be synchronized and visible to game clients. It operates purely on entity creation.

*   **EntityAdded:** A reactive system that triggers when a marker entity is fully added to the world. Its primary function is to link the new marker entity with any existing, active ReachLocationTask instances. It queries the global ObjectiveDataStore and calls the setupMarker method on relevant tasks, effectively "activating" the marker from the perspective of the quest logic.

*   **Ticking:** The primary runtime logic system. On each server tick, this system queries for all active markers. For each marker, it performs a spatial query to find all players within its defined radius. It maintains the state of which players are inside the radius and fires an event, onPlayerReachLocationMarker, only when a player *newly enters* the area. This prevents continuous event firing for players who remain inside the marker's bounds.

### Lifecycle & Ownership
- **Creation:** These systems are not instantiated directly by developers. They are discovered and instantiated by the server's ECS (Entity Component System) framework during the world or plugin loading phase.
- **Scope:** The system instances are singletons within the context of a running world. They persist for the entire duration of a server session.
- **Destruction:** The systems are destroyed and cleaned up when the server world is unloaded or during a full server shutdown. There is no manual destruction process.

## Internal State & Concurrency
- **State:** The container class ReachLocationMarkerSystems is stateless. The nested systems are also designed to be stateless, operating exclusively on the component data of the entities they process in each execution cycle.
- **Thread Safety:** The Ticking system is explicitly marked as **not thread-safe** and will not be parallelized by the ECS scheduler (isParallel returns false). This is due to its reliance on global, non-thread-safe resources like the ObjectiveDataStore and the spatial query system. It also uses a ThreadLocal cache for UUID sets to optimize performance by reducing memory allocations, a pattern that is inherently single-threaded per system execution.

**WARNING:** Modifying the ECS scheduler to force parallel execution of the Ticking system will lead to race conditions, data corruption, and server instability.

## API Surface

The public contract for this class is defined by the behavior of its nested systems within the ECS framework, not by traditional callable methods. Developers interact with these systems indirectly by creating entities with specific component combinations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EnsureNetworkSendable | HolderSystem | O(1) | System. Reacts to entity creation to add a NetworkId component. |
| EntityAdded | RefSystem | O(N) | System. Reacts to entity creation to link it to N active objective tasks. |
| Ticking | EntityTickingSystem | O(M*log(P)) | System. Runs every tick. For each of M markers, performs a spatial query against P players. |

## Integration Patterns

### Standard Usage

A developer never interacts with ReachLocationMarkerSystems directly. The correct pattern is to create an entity and attach the ReachLocationMarker and TransformComponent components. The systems will automatically detect and manage this entity.

```java
// Example of creating a marker entity, which these systems will then process.
// This code would exist within a separate game logic system.

// Get a command buffer from the ECS context
CommandBuffer<EntityStore> commands = ...;

// Create a new entity
Ref<EntityStore> markerEntity = commands.createEntity();

// Add the transform to position it in the world
TransformComponent transform = new TransformComponent();
transform.setPosition(new Vector3d(100, 64, 200));
commands.addComponent(markerEntity, transform);

// Add the marker component to activate the systems' logic
ReachLocationMarker marker = new ReachLocationMarker("unique_marker_id_from_asset");
commands.addComponent(markerEntity, marker);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create instances of the nested systems (e.g., `new Ticking(...)`). The ECS framework is solely responsible for their lifecycle. Manual instantiation will result in non-functional, rogue system objects that are not registered with the game loop.
- **Manual Execution:** Do not call the `tick` or `onEntityAdded` methods directly. These are callbacks managed by the ECS scheduler and should only be invoked by the engine.
- **Missing Dependencies:** Creating a marker entity without a TransformComponent will cause the EntityAdded and Ticking systems to ignore it, as their queries will not match. The marker will exist but will be non-functional.

## Data Pipeline

The flow of data and control is dictated by entity state changes and the server game loop.

> **Marker Creation Flow:**
> Entity is created with `ReachLocationMarker` -> `EnsureNetworkSendable` adds `NetworkId` -> `EntityAdded` queries `ObjectiveDataStore` -> `ReachLocationTask.setupMarker()` is called.

> **Player Interaction Flow (Per Tick):**
> Game Tick -> **Ticking System** -> Spatial Query for nearby players -> Compare with previous state -> If new player entered -> `ObjectiveDataStore` is queried -> `ReachLocationTask.onPlayerReachLocationMarker()` is called -> Objective state is updated.


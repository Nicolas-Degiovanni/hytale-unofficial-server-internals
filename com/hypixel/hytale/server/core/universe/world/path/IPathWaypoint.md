---
description: Architectural reference for IPathWaypoint
---

# IPathWaypoint

**Package:** com.hypixel.hytale.server.core.universe.world.path
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface IPathWaypoint {
```

## Architecture & Concepts
The IPathWaypoint interface defines the contract for a single, discrete point within a server-side entity path. It serves as a fundamental data structure for the AI navigation and scripted event systems. Architecturally, it decouples the concept of a "path" from the specific implementation of its waypoints, allowing for both static and highly dynamic navigation points.

The most critical design feature is the indirection provided by the `getWaypointPosition` and `getWaypointRotation` methods. By accepting a ComponentAccessor for the EntityStore, a waypoint's location is not required to be a fixed, pre-calculated coordinate. Instead, it can be resolved dynamically at runtime. This enables sophisticated AI behaviors, such as:
- Patrolling to the location of another entity.
- Following a moving platform.
- Moving to a location defined by a game script or player interaction.

This interface is a core component of the server's AI Behavior Tree or State Machine, where a "Follow Path" task would consume a sequence of IPathWaypoint objects to direct an entity's movement.

## Lifecycle & Ownership
As an interface, IPathWaypoint itself has no lifecycle. The lifecycle and ownership semantics apply to its concrete implementations.

- **Creation:** Instances are typically created and aggregated into a list or sequence by a higher-level system, such as a `PathFactory` or a script parser, when an entity's path is defined or assigned. This can occur at world-generation time for static paths or dynamically in response to game events.
- **Scope:** The lifetime of a waypoint implementation is tied to the lifetime of the path it belongs to. It persists as long as the path is relevant to an AI controller or scripted sequence.
- **Destruction:** Waypoints are eligible for garbage collection when the containing `Path` object is dereferenced. This typically happens when an entity completes its path, is destroyed, or has its AI state reset.

## Internal State & Concurrency
- **State:** The interface defines no state. Implementations are strongly recommended to be immutable. If an implementation must contain mutable state, it is the responsibility of the implementer to ensure thread safety. The primary state is resolved *on-demand* via the ComponentAccessor, not stored within the waypoint itself.
- **Thread Safety:** The interface itself is thread-safe. However, implementations are only as thread-safe as their design allows. The methods accept a ComponentAccessor, which provides safe, read-only access to the world state for the duration of the call. Implementations must not attempt to store or mutate the ComponentAccessor.

**WARNING:** Calls to resolve waypoint positions must be executed within the server's main update tick to guarantee a consistent and safe view of the EntityStore. Accessing it from an asynchronous task without proper synchronization will lead to severe concurrency violations and world corruption.

## API Surface
The public contract of IPathWaypoint is designed for querying by an AI path-following system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOrder() | int | O(1) | Returns the index of this waypoint in a sequence. Used for ordering. |
| getWaypointPosition(accessor) | Vector3d | Variable | **Core Method.** Dynamically resolves the world-space position of the waypoint. Complexity depends on implementation (e.g., entity lookup). |
| getWaypointRotation(accessor) | Vector3f | Variable | Dynamically resolves the required entity rotation at the waypoint. |
| getPauseTime() | double | O(1) | The duration in seconds an entity should wait upon reaching this waypoint. |
| getObservationAngle() | float | O(1) | The yaw angle an entity should face while paused at this waypoint. |

## Integration Patterns

### Standard Usage
An AI behavior, such as a `PathFollowerTask`, retrieves the current waypoint from its path sequence and uses it to update the entity's navigation target each tick.

```java
// Simplified example within an AI task
IPathWaypoint currentWaypoint = entity.getPath().getCurrentWaypoint();
ComponentAccessor<EntityStore> accessor = world.getEntities();

// Resolve the target position for this frame
Vector3d targetPosition = currentWaypoint.getWaypointPosition(accessor);
entity.getNavigationComponent().setTarget(targetPosition);

// Check for arrival and pause if necessary
if (entity.getPosition().distance(targetPosition) < 0.5) {
    double pause = currentWaypoint.getPauseTime();
    if (pause > 0) {
        entity.getAI().pause(pause);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Position Caching:** Do not cache the result of `getWaypointPosition`. The core architectural benefit is its dynamic nature. Caching the position will break any behaviors that rely on a moving target.
- **Stateful Implementations:** Avoid creating implementations of IPathWaypoint that store mutable state. This can lead to unpredictable behavior if the same path is shared by multiple entities.
- **Ignoring Accessor:** Do not implement `getWaypointPosition` to return a static, pre-stored vector while ignoring the provided ComponentAccessor. This defeats the purpose of the design and should be handled by a more specialized, lightweight data structure if truly static coordinates are needed.

## Data Pipeline
IPathWaypoint acts as a dynamic data provider within the server's AI control flow. It does not process a stream of data but rather responds to on-demand queries.

> Flow:
> AI Behavior System -> Requests target position from current **IPathWaypoint** -> Implementation queries **EntityStore** via ComponentAccessor -> Returns resolved Vector3d -> AI Behavior System -> Updates Entity Navigation Component


---
description: Architectural reference for IPathProvider
---

# IPathProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IPathProvider extends ExtraInfoProvider {
```

## Architecture & Concepts
The IPathProvider interface defines a standardized contract for any component that can hold and expose pathfinding data for a server-side entity, typically an NPC. It serves as a data-centric component within the AI sensory system, abstracting the underlying pathfinding logic from the navigation and behavior logic.

By extending ExtraInfoProvider, this interface integrates into a larger, type-safe component system. This allows various AI systems, such as sensors or behavior trees, to query an entity for its pathing capabilities without being tightly coupled to a specific implementation. The core responsibility of an IPathProvider implementation is to act as a stateful container for a calculated path, representing the intended route for an NPC to its destination.

## Lifecycle & Ownership
As an interface, IPathProvider itself does not have a lifecycle. The lifecycle described here pertains to its concrete implementations.

- **Creation:** An implementation of IPathProvider is typically instantiated and attached to an NPC entity during the entity's own initialization. It is created alongside other core AI components.
- **Scope:** The component's lifetime is strictly bound to its parent NPC entity. It persists as long as the NPC is active in the world.
- **Destruction:** The component is marked for garbage collection when its parent NPC is despawned or destroyed. The `clear` method may be invoked as part of the entity's cleanup routine to release references to path data.

## Internal State & Concurrency
- **State:** Implementations are inherently stateful and mutable. They hold a reference to an IPath object, which can be set by a pathfinding system and invalidated via the `clear` method. The state directly reflects whether the associated NPC has a valid, active path to follow.
- **Thread Safety:** **Not guaranteed.** Pathfinding calculations may occur on background threads, while path consumption by the navigation system occurs on the main server thread. Implementations of this interface **must** be designed for concurrent access. It is critical that writes to the internal path state (from a pathfinding thread) and reads (from the main game loop) are properly synchronized to prevent partial reads or race conditions. Failure to ensure thread safety will lead to severe server instability and unpredictable NPC behavior.

## API Surface
The public contract is minimal, focusing exclusively on querying and clearing path state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasPath() | boolean | O(1) | Checks for the existence of a valid path. This should always be checked before calling getPath. |
| getPath() | IPath | O(1) | Returns the current path object, or null if no path is available. |
| clear() | void | O(1) | Invalidates and removes the current path. This is used when a path is completed, becomes invalid, or the NPC's goal changes. |
| getType() | Class | O(1) | Returns the class literal for IPathProvider, enabling type-safe retrieval from the ExtraInfoProvider system. |

## Integration Patterns

### Standard Usage
The primary consumer is the NPC's navigation or movement system. On each tick, it queries the provider to guide the entity's movement.

```java
// Within an NPC's AI update loop
IPathProvider pathProvider = npc.getInfo(IPathProvider.class);

if (pathProvider != null && pathProvider.hasPath()) {
    IPath<?> path = pathProvider.getPath();
    // ... logic to follow the next waypoint in the path
} else {
    // ... logic to request a new path or stand idle
}
```

### Anti-Patterns (Do NOT do this)
- **Path Caching:** Do not retrieve the IPath object once and store it long-term in another system. The path held by the provider can be cleared at any moment. Always re-query the provider using `hasPath` and `getPath` on each update cycle.
- **Assuming Non-Null:** Never call `getPath` without first checking `hasPath`. Doing so may result in a NullPointerException if the NPC has no active path.
- **Direct Modification:** Do not attempt to modify the IPath object returned by `getPath`. The interface does not guarantee that the returned object is mutable or that modifications will be respected by the navigation system. Treat the returned path as a read-only data structure.

## Data Pipeline
IPathProvider is a critical junction in the NPC pathfinding data flow, acting as the handoff point between path calculation and path execution.

> Flow:
> AI Goal System -> Pathfinding Service Request -> Path Calculation -> IPath Object -> **IPathProvider Implementation** -> NPC Navigation System -> Entity Transform Update


---
description: Architectural reference for AStarDebugWithTarget
---

# AStarDebugWithTarget

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Transient Utility

## Definition
```java
// Signature
public class AStarDebugWithTarget extends AStarDebugBase {
```

## Architecture & Concepts
The AStarDebugWithTarget class is a concrete implementation of the Decorator pattern, designed exclusively for debugging server-side NPC pathfinding. It wraps an instance of AStarWithTarget, the core pathfinding algorithm, to provide enhanced, context-aware diagnostic output.

This class does not implement any pathfinding logic itself. Instead, it intercepts specific stages of the pathfinding process—delegated by its parent, AStarDebugBase—to inject target-specific information into logs and visual map dumps. Its primary contribution is to orient the debugging output around the pathfinding *destination*, plotting the target's location and calculating metrics relative to it.

This component is a development tool. It is critical to understand that it is not part of the core navigation engine and must be disabled in production environments to prevent significant performance degradation.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level navigation system, typically triggered by a server-side debug flag or developer command. It is created to wrap a freshly instantiated AStarWithTarget object before a pathfinding calculation begins.
- **Scope:** Its lifetime is ephemeral, strictly bound to the single pathfinding operation it is debugging. It exists only for the duration of one path calculation.
- **Destruction:** The object is eligible for garbage collection immediately after the pathfinding result is computed and its diagnostic output has been logged. It manages no persistent resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is stateless on its own. It holds a final reference to the AStarWithTarget instance it decorates. All state it interacts with belongs to the wrapped pathfinder object, which is highly mutable during the path calculation.
- **Thread Safety:** **Not Thread-Safe.** Pathfinding operations are synchronous and computationally intensive, expected to be executed exclusively on a single thread responsible for a given NPC's logic update. Accessing this class or the object it wraps from multiple threads will lead to state corruption and unpredictable behavior.

## API Surface
The public API is inherited from AStarDebugBase. The methods below are protected overrides that implement the specific debugging contract for target-aware pathfinding.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDumpMapRegionZ(int) | int | O(1) | Overrides base behavior to return the Z-coordinate of the target's position. |
| getDumpMapRegionX(int) | int | O(1) | Overrides base behavior to return the X-coordinate of the target's position. |
| drawMapFinish(StringBuilder[], int, int) | void | O(N) | Hooks into the map rendering process to plot a special character (Ω) at the target's final location. |
| getExtraLogString(MotionController) | String | O(1) | Appends target-specific data, like the destination coordinates and estimated distance, to the debug log line. |

## Integration Patterns

### Standard Usage
This class should be conditionally applied to a core AStarWithTarget instance when debugging is required. The navigation system should treat the wrapper and the core object interchangeably through the AStarBase interface.

```java
// A navigation service decides whether to enable debugging
AStarWithTarget corePathfinder = new AStarWithTarget(world, start, target);
AStarBase pathfinder;

if (server.isDebugModeEnabled()) {
    // Wrap the core pathfinder with the debug decorator
    pathfinder = new AStarDebugWithTarget(corePathfinder, server.getLogger());
} else {
    pathfinder = corePathfinder;
}

// The system proceeds, unaware of the specific implementation
PathResult result = pathfinder.findPath();
```

### Anti-Patterns (Do NOT do this)
- **Production Instantiation:** Never create an AStarDebugWithTarget in a production environment. The overhead of string formatting and map generation for every pathfinding request is substantial and will severely impact server performance.
- **State Leakage:** Do not retain a reference to this object after the pathfinding operation is complete. Its lifecycle is intentionally short, and holding onto it can prevent the wrapped AStarWithTarget and its cached data from being garbage collected.

## Data Pipeline
AStarDebugWithTarget does not participate in the primary data pipeline of pathfinding. Instead, it acts as an inspector, tapping into the state of the core algorithm to generate a secondary stream of diagnostic data.

> **Debug Information Flow:**
> Pathfinding Request -> `AStarWithTarget` (Internal State Update) -> **`AStarDebugWithTarget`** (Reads State) -> Formats Debug String & Map -> `HytaleLogger` -> Log File / Console


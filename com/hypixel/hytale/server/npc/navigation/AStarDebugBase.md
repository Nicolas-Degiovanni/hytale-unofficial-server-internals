---
description: Architectural reference for AStarDebugBase
---

# AStarDebugBase

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Utility

## Definition
```java
// Signature
public class AStarDebugBase {
```

## Architecture & Concepts
AStarDebugBase is a server-side diagnostic utility designed to provide a human-readable visualization of the internal state of an AStarBase pathfinding process. It is not part of the core pathfinding algorithm itself; rather, it acts as an external inspector or renderer that translates the abstract data structures of a pathfinder—such as the open set, closed set, and final path—into formatted text logs.

The primary architectural role of this class is to decouple debugging and logging concerns from the high-performance pathfinding logic in AStarBase. By injecting an AStarBase instance, it can read its state without polluting the core algorithm with diagnostic code.

The class is designed for extension, employing a Template Method pattern through its protected, empty-body methods like drawMapFinish and getExtraLogString. This allows for specialized debuggers to be created for different entity types or scenarios by overriding these hooks to provide more context-specific information.

## Lifecycle & Ownership
- **Creation:** An AStarDebugBase instance is created manually and on-demand, typically within a debug command handler or a temporary diagnostic block. It is constructed with a direct reference to the live AStarBase instance that needs to be inspected. It is not managed by a service container or registry.
- **Scope:** The intended lifetime is extremely short and ephemeral. An instance should only exist for the duration of the debugging operation. It is created, one of its dump methods is called, and it should then be immediately eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no explicit cleanup methods. Holding a reference to an AStarDebugBase instance will also prevent its associated AStarBase object from being collected, which can lead to memory leaks if not managed carefully.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It holds immutable references to an AStarBase instance and a HytaleLogger. All complex data structures, such as the character grid for the map visualization, are created as local variables within method scopes and do not persist between calls.
- **Thread Safety:** **This class is not thread-safe.** It performs no internal locking and directly iterates over the collections within the provided AStarBase instance. It is critically assumed that the pathfinding algorithm is paused or that the call to a dump method is made from the same thread that is executing the pathfinding logic. Accessing this class from a separate thread while the AStarBase state is being modified will result in `ConcurrentModificationException` or, worse, silent data corruption and inconsistent debug output. All synchronization must be handled externally.

## API Surface
The public API is focused exclusively on triggering different formatted dumps to the logger.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| dumpOpens(controller) | void | O(N) | Logs a detailed list of all nodes currently in the A* "open set". N is the size of the open set. |
| dumpPath() | void | O(P) | Logs the sequence of nodes that form the final, calculated path. P is the length of the path. |
| dumpMap(drawPath, controller) | void | O(V) | Renders and logs a 2D text-based map of the entire search area, including open, closed, and path nodes. V is the total number of visited nodes. |

## Integration Patterns

### Standard Usage
The class is intended to be instantiated for a single inspection operation and then discarded. It requires a live AStarBase instance and a logger.

```java
// Given an active pathfinder instance 'npcPathfinder' and a 'serverLogger'
// This code might exist inside a server command for debugging an NPC.

AStarDebugBase debugger = new AStarDebugBase(npcPathfinder, serverLogger);

// Dumps a complete visual map of the pathfinder's current state.
debugger.dumpMap(true, npcMotionController);
```

### Anti-Patterns (Do NOT do this)
- **Long-Lived Instances:** Do not cache or store instances of AStarDebugBase in fields. This will prevent the AStarBase instance and its potentially large internal node maps from being garbage collected, leading to a memory leak.
- **Cross-Thread Inspection:** Do not create an AStarDebugBase instance on a main thread and pass it a pathfinder that is being actively modified on a worker thread. This is a severe race condition.

## Data Pipeline
AStarDebugBase functions as a data sink for visualization, not as a processing stage in a pipeline. Its flow is a one-way "snapshot" of another component's state.

> Flow:
> AStarBase Internal State (Node Maps, Lists) -> **AStarDebugBase** (Reads and Formats) -> HytaleLogger -> Log Output (Console/File)


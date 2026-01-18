---
description: Architectural reference for NPCBlackboardCommand
---

# NPCBlackboardCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Utility

## Definition
```java
// Signature
public class NPCBlackboardCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The NPCBlackboardCommand class serves as the primary administrative and diagnostic entry point for the server-side NPC Blackboard system. It is not a component of active gameplay logic; rather, it is a collection of console commands designed for developers and server operators to inspect, debug, and manipulate the internal state of the Blackboard.

This class acts as a command aggregator, registering a suite of nested subcommands that each target a specific aspect of the Blackboard's data. The Blackboard itself is a critical performance optimization for the NPC system. It functions as a shared, world-aware cache, allowing NPCs to query for environmental information (e.g., locations of specific block types, nearby entities of interest, resource availability) without performing expensive, real-time world scans. This command collection provides the necessary tools to look "under the hood" of that cache.

Each subcommand queries the core Entity Component System (ECS) stores—specifically the EntityStore and ChunkStore—and the global Blackboard resource to retrieve and format its state for human-readable output.

## Lifecycle & Ownership
- **Creation:** An instance of NPCBlackboardCommand is created once by the server's command registration system during the bootstrap phase, typically when the NPCPlugin is loaded. The constructor immediately registers all its nested subcommand classes.
- **Scope:** The object persists for the entire server session. It is held in memory by the central command registry, ready to dispatch incoming command requests.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only upon server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** The NPCBlackboardCommand class and its nested subcommand classes are fundamentally stateless. They do not maintain any mutable state across executions. All state they interact with is external, residing within the World, the ECS stores, and the Blackboard resource itself.
- **Thread Safety:** This command system is not thread-safe and must not be invoked from any thread other than the main server thread. The Hytale server architecture guarantees that all command executions occur synchronously on the main game loop tick. Any attempt to access the underlying ECS stores or Blackboard resource from an asynchronous task will lead to severe concurrency violations, data corruption, and server instability.

## API Surface
The public contract of this class is not a programmatic API but the set of server commands it exposes. Direct invocation of its methods is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| chunks | Subcommand | O(N) | Lists all chunk sections that contain Blackboard data. Complexity is proportional to the number of tracked chunk sections. |
| chunk [position] | Subcommand | O(1) | Dumps detailed Blackboard data for the chunk at the specified block position. |
| drop | Subcommand | O(M) | **Destructive.** Clears all data from the entire NPC Blackboard resource. Complexity is proportional to the number of views. |
| views | Subcommand | O(M) | Lists all active BlockTypeViews, which aggregate block data over large regions. Complexity is proportional to the number of views. |
| view [chunk] | Subcommand | O(1) | Dumps detailed data for the BlockTypeView corresponding to the specified chunk position. |
| blockevents | Subcommand | O(E) | Dumps the state of the BlockEventView, showing which NPCs are listening for events related to specific BlockSets. |
| entityevents | Subcommand | O(E) | Dumps the state of the EntityEventView, showing which NPCs are listening for events related to specific NPCGroups. |
| resourceviews | Subcommand | O(M) | Lists all active ResourceViews, which track resource locations and reservations. |
| resourceview [chunk] | Subcommand | O(1) | Dumps detailed data for the ResourceView at the specified chunk position, including reservations. |
| reserve [entity] [true/false] | Subcommand | O(1) | Manually adds or removes a reservation on an NPC for the command sender (player). |
| reservation [entity] | Subcommand | O(1) | Checks the reservation status of a target NPC with respect to the command sender. |

## Integration Patterns

### Standard Usage
This class is intended for use by server administrators via the server console or in-game chat. It is not meant to be integrated into other game logic systems.

```java
// Example usage from in-game chat or server console
/npc blackboard chunks
/npc blackboard view ~ ~
/npc blackboard drop
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new NPCBlackboardCommand()`. The server's command system manages its lifecycle. Manual instantiation will result in a non-functional object that is not registered to handle commands.
- **Direct Method Invocation:** Do not call the `execute` method on any of the nested subcommand classes directly. This bypasses the entire command processing pipeline, including argument parsing, permission checks, and context setup, and will fail.
- **Gameplay Logic Dependency:** Do not build gameplay features that rely on this class. It is a debugging tool. Its functionality, command names, and output are subject to change without notice and are not considered a stable API for game mechanics.

## Data Pipeline
The data flow for any command in this collection follows a standard server command processing pipeline.

> Flow:
> User Input (Console/Chat) -> Server Network Layer -> Command Parser -> **NPCBlackboardCommand Dispatcher** -> Subcommand Execution -> Read from ECS Stores & Blackboard Resource -> Format `Message` Object -> Send to Client/Console

---

